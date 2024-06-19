<script setup>
  import { PDFDocument, rgb } from 'pdf-lib'
  import fontkit from '@pdf-lib/fontkit'
  import { ref, onMounted, reactive, computed } from 'vue'
  import { CircleCloseFilled } from '@element-plus/icons-vue'
  import { ElMessage, ElMessageBox } from 'element-plus'
  import $ from 'jquery'
  import YouSheBiaoTiHei from '@/assets/fonts/YouSheBiaoTiHei.ttf'
  import { EditPen, Text, Square, Withdraw, SaveAll, SaveEdit } from '@/assets/svg/icon'
  
  let count = ref(0)
  let pdfDoc = reactive({})
  let total =  ref(0)
  let pdfUrl = ref('http://storage.xuetangx.com/public_assets/xuetangx/PDF/PlayerAPI_v1.0.6.pdf')
  let currentPage = ref(1)
  let operationalBehaviorAll = reactive({})
  let freehandLines = computed(() => {
    let currentData = operationalBehaviorAll[currentPage.value] || []
    return currentData.filter((item) => item.operation === 'line')
  })
  let contentList = computed(() => {
    let currentData = operationalBehaviorAll[currentPage.value] || []
    return currentData.filter((item) => item.operation === 'text')
  })
  let squareList = computed(() => {
    let currentData = operationalBehaviorAll[currentPage.value] || []
    return currentData.filter((item) => item.operation === 'square')
  })
  let pdfHeight = ref(0)
  let pdfSize = reactive({ width: '0', height: '0' })
  let pdfContent = ref(null)
  let editingMethod = ref('')
  let drawing = ref(false)
  let node = reactive({})
  // 操作选项
  const editingOptions = [
    { icon: EditPen, method: 'drawing', behavior: 'drawingBoard', tooltip: '画线' },
    { icon: Text, method: 'text', behavior: 'drawingBoard', tooltip: '文字' },
    { icon: Square, method: 'square', behavior: 'drawingBoard', tooltip: '边框' },
    { icon: Withdraw, method: 'undo', behavior: 'system', tooltip: '撤回'},
    { icon: SaveAll, method: 'saveAll', behavior: 'system', tooltip: '保存全部'},
    { icon: SaveEdit, method: 'saveEdit', behavior: 'system', tooltip: '保存有编辑痕迹的文件'},
  ]
  //打字板
  let inputContent = reactive({})
  let inputShow = ref(false)
  let moveLoading = ref(false)
  let inputPosition = reactive({ top: '0px', left: '0px' })
  // 颜色选择器
  const lineColor = ref('#F90303')
  // 初始化
  function init() {
    var url = pdfUrl.value + '?' + Math.random()// 去除缓存
    getPdfContent(url);
  }
  async function getPdfContent(url) { // 加载pdf，并分页
    const arrayBuffer = await fetch(url, { mode: 'cors' }).then((res) => res.arrayBuffer())
    pdfDoc = await PDFDocument.load(arrayBuffer)
    total.value = pdfDoc.getPages().length// 总页数
    modifyPdf(currentPage.value)// 显示第一页
  }
  // 加载pdf
  async function modifyPdf (p) {
    currentPage.value = p
    closeEdit()
    const page = pdfDoc.getPage(p * 1 - 1)
    const { width, height } = page.getSize()
    pdfHeight.value = height
    pdfSize.width = `${width}`
    pdfSize.height = `${height}`
    const pdfTempDoc = await PDFDocument.create()// pdf的显示
    const copiedPages = await pdfTempDoc.copyPages(pdfDoc, [p * 1 - 1])
    pdfTempDoc.addPage(copiedPages[0])
    if (window.navigator && window.navigator.msSaveOrOpenBlob) {
      const blob = new Blob([await pdfTempDoc.save()], { type: 'application/pdf' })
      console.log(blob)
    } else {
      const pdfUrl = URL.createObjectURL(
        new Blob([await pdfTempDoc.save()], { type: 'application/pdf' })
      )
      pdfContent.value = pdfUrl
    }
  }
  // 改变操作方式
  function changeEditingMethod (param) {
    let { method, behavior } = param
    if (behavior === 'drawingBoard') {
      editingMethod.value = method
    }
    if (method === 'drawing') {
      $('#svgLine').off('mousedown mousemove mouseup')
    }
    if (method === 'square') {
      drawLineDone();
    }
    switch (method) {
      case 'undo':
        undo()
        break
      case 'saveAll':
        saveAll()
        break
      case 'saveEdit':
        saveEdit()
        break
    }
  }
  // 撤回
  function undo() {
    let currentFreehandLines = operationalBehaviorAll[currentPage.value]
    if (currentFreehandLines && currentFreehandLines.length) {
      currentFreehandLines.pop()
    } else {
      ElMessage({
        message: '当前页没有可撤回的操作',
        type: 'warning'
      })
    }
  }
  // 处理颜色
  function hexToRgb(hex) {
      // 去掉可能的 # 前缀
      hex = hex.replace(/^#/, '')
      // 解析RGB值
      const bigint = parseInt(hex, 16)
      const r = (bigint >> 16) & 255
      const g = (bigint >> 8) & 255
      const b = bigint & 255
  
      // return { r, g, b }
      return rgb(r / 255, g / 255, b / 255)
  }
  // 边框
  function drawLineDone() {
    $('#svgLine').mousedown(function(position) {
      const that = this
      const x1 = position.offsetX
      const y1 = position.offsetY
      saveBehaviorAll({
        page: currentPage.value,
        x: x1 * 1,
        y: y1 * 1,
        width: 0,
        height: 0,
        operation: 'square',
        color: lineColor.value,
        id: new Date().getTime() + Math.random()
      })
      $(this).mousemove(function(e) {
        let { offsetX, offsetY } = e
        let currentFreehandLines = operationalBehaviorAll[currentPage.value] || []
        let square = currentFreehandLines[currentFreehandLines.length - 1]
        const x = offsetX
        const y = offsetY
        const width = x - x1 * 1
        const height = y - y1 * 1
        if (width > 0) {
          square.width = width
        } else {
          square.width = -width
          square.x = x
          square.y = y
        }
        if (height > 0) {
          square.height = height
        } else {
          square.height = -height
          square.x = x
          square.y = y
        }
      })
      $(this).mouseup(function() {
        $(this).unbind('mousemove')
        $(this).unbind('mouseup')

        // 手动设置新矩形的颜色
        const newRect = document.querySelector('.svgrect:last-child')
        if (newRect) {
          newRect.style.stroke = lineColor.value
        }
      })
    })
  }
  // 写字板
  function activeEdit(e) {
    if (moveLoading.value || inputShow.value) {
      node = null
      closeEdit()
    } else if (editingMethod.value === 'text') {
      const { offsetX, offsetY } = e
      inputPosition.top = `${offsetY}px`
      inputPosition.left = `${offsetX}px`
      inputShow.value = true
      inputContent.color = lineColor.value,
      inputContent.top = `${offsetY}`
      inputContent.left = `${offsetX}`
      inputContent.operation = `text`
      inputContent.page = currentPage.value
      inputContent.id = new Date().getTime() + Math.random()
    }
  }
  function closeEdit() {
    inputShow.value = false
    inputContent.inputValue = null
  }
  function submitEdit() {
    saveBehaviorAll(JSON.parse(JSON.stringify(inputContent)))
    closeEdit()
  }
  // 修改文字标注
  function activeClick(d) {
    const { left, top } = d
    inputPosition.top = `${top}px`
    inputPosition.left = `${left}px`
    inputContent = JSON.parse(JSON.stringify(d))
    inputShow.value = true
  }
  // 文字拖动事件
  function moveDownPosition(position) {
    moveLoading.value = true
    const id = $(position.target).parents('.divbox').data('id')
    let currentData = operationalBehaviorAll[currentPage.value] || []
    node = (currentData.filter(item => Number(item.id) === Number(id)))[0]
  }
  function movePosition(position) {
    if (node) {
      // eslint-disable-next-line no-empty
      if ((node.left < 0 && position.movementX < 0) || (node.top < 0 && position.movementY < 0) || (node.left > pdfSize.width && position.movementX > 0) || (node.top > pdfSize.height && position.movementY > 0)) {
      } else {
        node.left = node.left * 1 + position.movementX * 1
        node.top = node.top * 1 + position.movementY * 1
      }
    }
  }
  function mouseupPosition(position) {
    node = null
    setTimeout(() => {
      moveLoading.value = false
    }, 500)
  }
  function removediv(d, index) {
    ElMessageBox.confirm(
    '确定要删除该标记吗?',
    '提示',
    {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'warning',
    }
  )
    .then(() => {
      let currentList = operationalBehaviorAll[currentPage.value]
      let temp = currentList.findIndex((val) => {
        if (val.id === d.id) {
          return val
        }
      })
      currentList.splice(temp, 1)
    })
    .catch(() => {})
  }
  // 自由线条
  function startDrawing(event) {
    if (editingMethod.value === 'drawing') {
      drawing.value = true
      const { offsetX, offsetY } = event
      // 新增：添加新的自由线条对象到数组
      saveBehaviorAll({
          pathData: `M ${offsetX} ${offsetY}`,
          color: lineColor.value,
          operation: 'line',
          id: new Date().getTime() + Math.random()
        })
    }
  }
   
  function draw(event) {
    if (drawing.value && editingMethod.value === 'drawing') {
      const { offsetX, offsetY } = event
      // 新增：更新最后一个自由线条对象的 pathData
      let currentFreehandLines = operationalBehaviorAll[currentPage.value]
      if (currentFreehandLines.length) {
        currentFreehandLines[currentFreehandLines.length - 1].pathData += ` L ${offsetX} ${offsetY}`
      }
    }
  }
   
  function stopDrawing() {
    if (editingMethod.value === 'drawing') {
      drawing.value = false
    }
  }
  // 统一记录操作
  function saveBehaviorAll(behavior) {
    if (operationalBehaviorAll[currentPage.value]) {
        let currentData = operationalBehaviorAll[currentPage.value] || []
        let index = currentData.findIndex(i => i.id === behavior.id)
        if (index !== -1) {
          // 修改文字标注
          currentData.splice(index, 1, behavior)
        } else {
          operationalBehaviorAll[currentPage.value].push(behavior)
        }
      } else {
        operationalBehaviorAll[currentPage.value] = [behavior]
      }
  }
  //
  async function generateLine() {
    let generatePdfDoc = await pdfDoc.copy()
    const pages = generatePdfDoc.getPages()
    const fontBytes = await fetch(YouSheBiaoTiHei).then((res) => res.arrayBuffer())// 添加字体包
    generatePdfDoc.registerFontkit(fontkit)
    const ubuntuFont = await generatePdfDoc.embedFont(fontBytes, { subset: true })// 不加subset:true,pdf会变得很大
    for (let k = 0; k < pages.length; k++) {
      let generatePage = pages[k]
      // 画线
      for (let i in operationalBehaviorAll) {
        if (Number(i) === k+1) {
          operationalBehaviorAll[i].forEach(element => {
            if (element.operation === 'line') {
              generatePage.drawSvgPath(element.pathData, {
                x: 0, y: pdfHeight.value * 1,
                borderColor: hexToRgb(element.color), // 红色
                borderWidth: 1,
              })
            } else if (element.operation === 'text') {
              generatePage.drawText(element.inputValue, {
                x: element.left * 1,
                y: pdfHeight.value * 1 - element.top * 1,
                size: 14,
                font: ubuntuFont,
                color: hexToRgb(element.color)
              })
            } else if (element.operation === 'square') {
              generatePage.drawRectangle({
                x: element.x * 1,
                y: pdfHeight.value * 1 - element.y * 1,
                width: element.width * 1,
                height: -element.height * 1,
                borderWidth: 2,
                borderColor: hexToRgb(element.color),
                opacity: 0,
                borderOpacity: 1
              })
            }
          });
        }
      }
    }
    return generatePdfDoc
  }
    // 保存所有
  async function saveAll() {
    let generatePdfDoc = await generateLine()
    downloadFile(generatePdfDoc)
  }
  async function saveEdit() {
    let generatePdfDoc = await generateLine()
    const pdfDoc = await PDFDocument.create()
    let behaviorPages = Object.keys(operationalBehaviorAll)
    if (!behaviorPages.length) {
      ElMessage({
        message: '没有编辑痕迹',
        type: 'warning'
      })
      return
    } else {
      const copiedPages = await pdfDoc.copyPages(generatePdfDoc, behaviorPages.map(item => {
        return item * 1 - 1
      }))
      copiedPages.forEach(item => {
        pdfDoc.addPage(item)
      })
      downloadFile(pdfDoc)
    }
  }
  // 下载文件
  async function downloadFile(PdfDoc) {
    const pdfBytes = await PdfDoc.save()
    const blob = new Blob([pdfBytes], { type: 'application/pdf' })
    const downLoadLink = document.createElement('a');
    downLoadLink.download = `NewPDF${new Date().getTime()}`;
    downLoadLink.href = URL.createObjectURL(blob);
    downLoadLink.click();
  }
  onMounted(() => {
    init()
  })
</script>

<template>
    <div class="pdf-editor">
      <!-- 操作区 -->
      <div class="operation-area">
        <el-pagination background layout="prev, pager, next" 
        :total="total" 
        :current-page="currentPage" 
        :page-size="1" 
        @current-change="d => (page=d) && modifyPdf(d)" />
        <div class="choosed-box">
          <div class="choose-color">
            <el-color-picker v-model="lineColor" />
          </div>
          <div class="choose-line">
            <el-tooltip
              class="box-item"
              effect="dark"
              :content="item.tooltip"
              placement="top-start"
              v-for="item in editingOptions" 
              :key="item.method" 
            >
            <!-- <el-icon
                :size="30"
                :color="editingMethod === item.method?'#077CF4':'#fff'"
                class="editor-icon"
                @click="changeEditingMethod(item)">
                  <component :is="item.icon" />
                </el-icon> -->
                <div :class="editingMethod === item.method ? 'active' : '' " 
                @click="changeEditingMethod(item)"
                class="svg-icon" v-html="item.icon"></div>
            </el-tooltip>
          </div>
        </div>
      </div>
      <!-- pdf显示区 -->
      <div class="view-box" :style="`width:${pdfSize.width}px;height:${pdfSize.height}px;user-select:none`">
        <iframe v-if="pdfContent !== null" ref="pdfViewer" :src="`${pdfContent}#scrollbars=0&toolbar=0&statusbar=0`" width="100%" height="100%" />
        <div class="board" @click.stop="activeEdit($event)" 
        @mousedown="moveDownPosition" 
        @mousemove="movePosition" 
        @mouseup="mouseupPosition">
          <!-- 文字标注 -->
          <div v-for="(d,index) in contentList" :key="index" 
          :data-id="d.id" 
          class="divbox" 
          :style="`position:absolute;z-index:3;left:${d.left-6}px;top:${d.top-8}px;color:${d.color}`" 
          @click.stop="" @dblclick.stop="activeClick(d)">
            <el-icon @click.stop="removediv(d,index)"
            class="el-icon-circle-close closediv">
              <component :is="CircleCloseFilled" size="32" />
            </el-icon>
            <div v-html="d.inputValue" />
          </div>
          <svg id="svgRect" class="svgRect">
            <g>
              <rect v-for="(d,index) in squareList" :key="index" :width="d.width" :height="d.height" :x="d.x" :y="d.y" class="svgrect" />
            </g>
          </svg>
   
          <svg id="svgLine" class="svgLine" @mousedown.stop="startDrawing" @mousemove.stop="draw" @mouseup.stop="stopDrawing">
            <g>
              <path v-for="(line, index) in freehandLines" :key="index" :d="line.pathData" :stroke="line.color" stroke-width="2" fill="none" />
            </g>
          </svg>
        </div>
        <!-- 写字板 -->
        <div v-if="inputShow" :style="inputPosition" class="inputBox">
          <textarea v-model="inputContent.inputValue" :rows="10" style="width:300px;" />
          <div class="sureBtn">
            <el-button size="small" @click="closeEdit">取消</el-button>
            <el-button type="success" size="small" @click="submitEdit">确定</el-button>
          </div>
        </div>
      </div> 
    </div>
</template>

<style scoped>
.pdf-editor {
  width: 100%;
}
.operation-area {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 20px;
  background: #020523;
}
.choosed-box {
  display: flex;
}

  .view-box {
    position: relative;
    width: 100%;
    height: 600px;
    margin: 0px auto;
    padding: 10px 0px 20px;
  }
  .pdf-input {
    width: 100px;
    line-height: 20px;
    border: 1px solid #ccc;
    background: #eee;
    z-index: 3;
  }
  .inputBox {
    background: #fff;
    padding: 5px;
    border-radius: 5px;
    position: absolute;
    z-index: 3;
  }
  .sureBtn {
    text-align: right;
    margin-top: 10px;
  }
  .board {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    cursor: pointer;
  }
  .svgrect {
    stroke: rgb(237, 10, 10);
    stroke-width: 2;
    position: relation;
    fill-opacity: 0;
  }
  .svgLine {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    cursor: pointer;
    z-index: 1;
  }
   
  .svgRect {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    cursor: pointer;
    z-index: 1;
  }
  .divbox {
    cursor: move;
    border: 1px solid red;
    z-index: 3;
    padding: 5px;
    white-space: nowrap;
  }
  .movediv {
    position: absolute;
    top: -8px;
    left: -8px;
  }
  .closediv {
    position: absolute;
    top: -8px;
    right: -8px;
    cursor: pointer;
    background: #f64404;
    color: #fff;
    border-radius: 50%;
  }
   
  .back-box {
    position: absolute;
    top: 93px;
    right: 150px;
  }
  .choose-line {
    height: 40px;
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
    align-items: center;
  }
  .choose-line .svg-icon ::v-deep svg {
    width: 20px!important;
    height: 20px!important;
    fill: #FFF;
    cursor: pointer;
    margin: 0 4px;
  }
  .choose-line .active ::v-deep svg {
    fill: #0339da;
   }
  .el-icon {
    margin: 0px 6px;
  }
  .choose-color {
    height: 40px;
    display: flex;
    flex-direction: row;
    margin-right: 8px;
  }
   
  .button {
    position: relative;
    transition: all 0.3s ease-in-out;
    box-shadow: 0px 10px 20px rgba(0, 0, 0, 0.2);
    padding-block: 0.5rem;
    padding-inline: 1.25rem;
    background-color: #1E96E1;
    border-radius: 9999px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #ffff;
    gap: 10px;
    font-weight: bold;
    border: 3px solid #ffffff4d;
    outline: none;
    overflow: hidden;
    font-size: 15px;
  }
   
  .icon {
    width: 24px;
    height: 24px;
    transition: all 0.3s ease-in-out;
  }
   
  .button:hover {
    transform: scale(1.05);
    border-color: #fff9;
  }
   
  .button:hover .icon {
    transform: translate(4px);
  }
   
  .button:hover::before {
    animation: shine 1.5s ease-out infinite;
  }
   
  .button::before {
    content: "";
    position: absolute;
    width: 100px;
    height: 100%;
    background-image: linear-gradient(
      120deg,
      rgba(255, 255, 255, 0) 30%,
      rgba(255, 255, 255, 0.8),
      rgba(255, 255, 255, 0) 70%
    );
    top: 0;
    left: -100px;
    opacity: 0.6;
  }
   
  @keyframes shine {
    0% {
      left: -100px;
    }
   
    60% {
      left: 100%;
    }
   
    to {
      left: 100%;
    }
  }
   
  .delete-button {
    background-color: #1E96E1;
    color: #fff;
    font-size: 14px;
    border: 0.5px solid rgba(0, 0, 0, 0.1);
    padding-bottom: 8px;
    width: 60px;
    height: 65px;
    border-radius: 15px 15px 12px 12px;
    cursor: pointer;
    position: relative;
    will-change: transform;
    transition: all .1s ease-in-out 0s;
    user-select: none;
    /* Add gradient shading to each side */
    background-image: linear-gradient(to right, rgba(0, 0, 0, 0.8), rgba(0, 0, 0, 0)),
      linear-gradient(to bottom, rgba(0, 0, 0, 0.8), rgba(0, 0, 0, 0));
    background-position: bottom right, bottom right;
    background-size: 100% 100%, 100% 100%;
    background-repeat: no-repeat;
    box-shadow: inset -4px -10px 0px rgba(255, 255, 255, 0.4),
      inset -4px -8px 0px rgba(0, 0, 0, 0.3),
      0px 2px 1px rgba(0, 0, 0, 0.3),
      0px 2px 1px rgba(255, 255, 255, 0.1);
    transform: perspective(70px) rotateX(5deg) rotateY(0deg);
  }
   
  .delete-button::after {
    content: '';
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
    background-image: linear-gradient(to bottom, rgba(255, 255, 255, 0.2), rgba(0, 0, 0, 0.5));
    z-index: -1;
    border-radius: 15px;
    box-shadow: inset 4px 0px 0px rgba(255, 255, 255, 0.1),
      inset 4px -8px 0px rgba(0, 0, 0, 0.3);
    transition: all .1s ease-in-out 0s;
  }
   
  .delete-button::before {
    content: '';
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
    background-image: linear-gradient(to right, rgba(0, 0, 0, 0.8), rgba(0, 0, 0, 0)),
      linear-gradient(to bottom, rgba(0, 0, 0, 0.8), rgba(0, 0, 0, 0));
    background-position: bottom right, bottom right;
    background-size: 100% 100%, 100% 100%;
    background-repeat: no-repeat;
    z-index: -1;
    border-radius: 15px;
    transition: all .1s ease-in-out 0s;
  }
   
  .delete-button:active {
    will-change: transform;
    transform: perspective(80px) rotateX(5deg) rotateY(1deg) translateY(3px) scale(0.96);
    height: 64px;
    border: 0.25px solid rgba(0, 0, 0, 0.2);
    box-shadow: inset -4px -8px 0px rgba(255, 255, 255, 0.2),
      inset -4px -6px 0px rgba(0, 0, 0, 0.8),
      0px 1px 0px rgba(0, 0, 0, 0.9),
      0px 1px 0px rgba(255, 255, 255, 0.2);
    transition: all .1s ease-in-out 0s;
  }
   
  .delete-button::after:active {
    background-image: linear-gradient(to bottom,rgba(0, 0, 0, 0.5), rgba(255, 255, 255, 0.2));
  }
   
  .delete-button:active::before {
    content: "";
    display: block;
    position: absolute;
    top: 5%;
    left: 20%;
    width: 50%;
    height: 80%;
    background-color: rgba(255, 255, 255, 0.1);
    animation: overlay 0.1s ease-in-out 0s;
    pointer-events: none;
  }
   
  @keyframes overlay {
    from {
      opacity: 0;
    }
   
    to {
      opacity: 1;
    }
  }
   
  .delete-button:focus {
    outline: none;
  }
</style>
