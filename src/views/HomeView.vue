<script setup>
import { reactive,onMounted,ref } from 'vue';
import { Edit } from '@element-plus/icons-vue'
import Speech from 'speak-tts';
import { ElMessageBox, ElNotification } from 'element-plus';
import axios from 'axios';
let tableData = JSON.parse(window.localStorage.getItem('key'))?reactive(JSON.parse(window.localStorage.getItem('key'))):reactive([])
const Socket = new WebSocket('ws://localhost:8001');
let dialogFormVisible = ref(false)
const dialogTableVisible = ref(false)
const name = ref('');
const department = ref('');

// console.log(Socket);

const form = reactive({
  region: ''
})

// Socket.binaryType = 'arraybuffer'
Socket.onopen = function(){
  getcall();
  // Socket.send('1');
}

//获取排队信息函数
const getcall =async ()=>{
  //获取储存在localstorage中存储的信息
  let callRes = JSON.parse(window.localStorage.getItem('key'))
  //for循环遍历callRes获取科室
  for (const key in callRes) {
    
    // console.log(callRes[key].department);
    let gcl =await getCallList(callRes[key].department);
    //声明一个空数组用来存储对应科室的排队人姓名
    let arr = []
    //for循环遍历请求来的排队人信息
    for (const k in gcl.data.data) {
      if (gcl.data.data[k].state === 1) {
        arr.push(gcl.data.data[k].name)
      }
    }
    tableData[key].queveUp = arr
    callRes[key].queveUp = arr
    window.localStorage.setItem('key',JSON.stringify(callRes));

    
  }
}


Socket.onmessage = function(evt){
  let index = evt.data.indexOf("|");
  let result = evt.data.substr(index + 1,evt.data.length);
  let res = evt.data.substring(0, index);
  if (result) {
    for (const key in tableData) {
      if (tableData[key].department === result) {
        tableData[key].name = res
        getcall();


        //呼叫弹窗
        dialogTableVisible.value = true
        name.value = res;
        department.value = result
        //播放声音
        voice();
        setInterval(function(){
          dialogTableVisible.value = false
        }, 5000)
      }
    }
  }
  if (evt.data === '2') {
    getcall();
  }
}


const getCallList =async (department)=>{
  return await axios.get(`/webApi/registration/getlist/${department}`)
}

const options = ref([]);
//获取科室列表回调
const getDepartment = async ()=>{
    const res = await axios.get("/webApi/department/list");
    options.value = res.data.data
}
const getRegistration = async ()=>{
    const res = await axios.get(`/webApi/registration/getlist/${form.region}`)
    return res.data.data
  }



// for (const key in tableData) {
//     tableData[key].name = '';
//     tableData[key].queveUp = []
//   }
//添加科室回调
const setDepartment = async ()=>{
  const res =await getRegistration();
  let state = true
  if (res.length !== 0) {
    let queveUp = []
    for (const key in res) {
      queveUp[key] = res[key].name
    }
 
    if (tableData.length === 0) {
        tableData.push({department:form.region,name:'',queveUp})
        dialogFormVisible.value = false
    }else{
      
      for (const key in tableData) {
        if (tableData[key].department === form.region) {
          alert('科室已添加，请重新选择！！')
          state = false
          break;
        }
      }
      if (state) {
        tableData.push({department:form.region,name:'',queveUp})
        dialogFormVisible.value = false
      }
    }
    
  }else{
    for (const key in tableData) {
        if (tableData[key].department === form.region) {
          alert('科室已添加，请重新选择！！')
          state = false
          break;
        }
      }
    if (state) {
      tableData.push({department:form.region,name:'',queveUp:''})
    }
    dialogFormVisible.value = false
  }
  
  window.localStorage.setItem('key',JSON.stringify(tableData));
  // console.log(JSON.parse(window.localStorage.getItem('key')));
}

onMounted(()=>{
  getDepartment();
})





//文字转语音
//定义一个源
const speech = new Speech();
//判断用户当前浏览器是否支持语音播报
if (speech.hasBrowserSupport()) {
  ElNotification({
    title: '语音提示',
    message: '支持语音合成',
    type: 'success',
    showClose: false,
  });
} else {
  ElNotification({
    title: '语音提示',
    message: '不支持语音合成',
    type: 'error',
    showClose: false,
  });
}
//初始化语音插件 - init内可以全部为空 - 自定义
speech
  .init({
    volume: 1, // 音量
    lang: 'zh-CN', // 语言
    rate: 1, // 语速
    pitch: 1, // 音调
    splitSentences: true, // 在句子结束时暂停
    listeners: {
      // 事件
      onvoiceschanged: voices => {
        // console.log('事件声音已更改', voices);
      },
    },
  })
  .then(data => {
    console.log('语音已准备好，声音可用', data);
  })
  .catch(e => {
    console.error('初始化时发生错误 : ', e);
  });
  //语音播报按钮
  function voice() {
  speech
    .speak({
      text: `请 ${name.value} 到 ${department.value} 科室就诊！`,
    })
    .then(() => {
      console.log('Success !');
    })
    .catch(e => {
      console.error('An error occurred :', e);
    });
}


</script>

<template>
  <header>
    <h1>排队等候</h1>
    <el-button class="but" type="primary" :icon="Edit" circle plain @click="dialogFormVisible = true"/>
  </header>
  <el-table :data="tableData" border stripe style="width: 100%">
    <el-table-column prop="department" label="科室" width="180" />
    <el-table-column prop="name" label="就诊中" width="180" />
    <el-table-column prop="queveUp" label="排队信息" />
  </el-table>

  <el-dialog v-model="dialogFormVisible" title="添加显示科室" width="500">
    <el-form :model="form">
      <el-form-item label="科室">
        <el-select v-model="form.region" placeholder="请选择科室">
          <el-option v-for="item in options" :label="item.name" :value="item.name" :key="item._id" />
        </el-select>
      </el-form-item>
    </el-form>
    <template #footer>
      <div class="dialog-footer">
        <el-button @click="dialogFormVisible = false">取消</el-button>
        <el-button type="primary" @click="setDepartment()">
          确定
        </el-button>
      </div>
    </template>
  </el-dialog>
  <el-dialog v-model="dialogTableVisible" title="呼叫" width="800">
    <h1>请 {{ name }} 到 {{ department }} 科室就诊</h1>
  </el-dialog>
</template>

<style lang="scss" scoped>
header{
  width: 100vw;
  height: 10vh;
  background-color: aqua;
  h1{
    margin: 0;
    text-align: center;
    height: 100%;
  }
  .but{
    position: absolute;
    top: 0;
    margin: 22px 8px;
  }
}
</style>