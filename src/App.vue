<script setup lang="ts">
import axios, { AxiosRequestConfig } from "axios";
import { onMounted, ref } from "vue";
const aaa = ref(null);
const bbb = ref(null);
const query = async () => {
  //没有access_token才需要登陆
  if (!localStorage.getItem("access_token")) {
    await login();
  }
  // 并发请求
  await [
    axios.get("http://localhost:3000/bbb"),
    axios.get("http://localhost:3000/bbb"),
    axios.get("http://localhost:3000/bbb"),
  ];

  const { data: aaaData } = await axios.get("http://localhost:3000/aaa");
  const { data: bbbData } = await axios.get("http://localhost:3000/bbb", {
    // headers:{
    //   Authorization: 'Bearer '+localStorage.getItem('access_token')
    // }
  });

  aaa.value = aaaData;
  bbb.value = bbbData;
};
onMounted(() => {
  query();
});
async function login() {
  const res = await axios.post("http://localhost:3000/user/login", {
    username: "mz",
    password: "123456",
  });
  localStorage.setItem("access_token", res.data.access_token);
  localStorage.setItem("refresh_token", res.data.refresh_token);
}

//拿到 access_token，然后在请求的时候带上
axios.interceptors.request.use(function (config) {
  const accessToken = localStorage.getItem("access_token");
  if (accessToken) {
    config.headers.authorization = "Bearer " + accessToken;
  }
  return config;
});

//token失效后自动刷新
async function refreshToken() {
  const res = await axios.get("http://localhost:3000/user/refresh", {
    params: {
      refresh_token: localStorage.getItem("refresh_token"),
    },
  });
  localStorage.setItem("access_token", res.data.access_token || "");
  localStorage.setItem("refresh_token", res.data.refresh_token || "");
  return res;
}
//定义类型
interface PendingTask{
  config:AxiosRequestConfig
  resolve:Function
}
//使并发多个请求时只刷新一次
let refreshing = false
let queue :PendingTask[]= []
axios.interceptors.response.use(
  (response) => {
    return response;
  },
  async (error) => {
    let { data, config } = error.response;
    console.log({ data, config },'@')

    if(refreshing){
      return new Promise((resolve)=>{
        queue.push({
          config,
          resolve
        })
        console.log(queue,'***')
      })
    }
    // 返回的错误是 401 就刷新 token，这里要排除掉刷新的 url，刷新失败不继续刷新
    if (data.statusCode === 401 && !config.url.includes("/user/refresh")) {
      refreshing = true
      const res = await refreshToken();
      refreshing = false;

      // 如果刷新接口返回的是 200，就用新 token 调用之前的接口
      if (res.status === 200) {
        queue.forEach(({config,resolve})=>{
          resolve(axios(config))
        })
        console.log(axios(config),'%%%')
        return axios(config);
      } else {
        alert("登录过期，请重新登录");
        return Promise.reject(res.data);
      }
    } else {
      return error.response;
    }
  }
);
</script>

<template>
  <div>
    <p>{{ aaa }}</p>
    <p>{{ bbb }}</p>
  </div>
</template>
