<template>
  <div v-if="isLoading" class="spinner-overlay">
    <div class="spinner"></div>
  </div>
  <div v-else class="container">
    <div class="top-buttons">
      <button class="save-btn btn" @click="save">SAVE</button>
      <button class="close-btn btn" @click="close">X</button>
    </div>
    
    <div class="section-title">Issue check</div>
    <div class="section-content">
      <div class="inline-elements">
        <label for="status">Status</label>
        <select id="status" v-model="status">
          <option value="" disabled>선택하세요</option>
          <option value="I">Improving</option>
          <option value="C">Improvement Completed</option>
          <option value="M">Monitoring</option>
          <option value="N">No Issue</option>
        </select>
      </div>
    </div>

    <div class="section-title">Issue Summary</div>
    <div class="section-content">
      <textarea id="issue-summary" 
      v-model="issueSummary" 
      placeholder="내용작성 (이슈칼럼임)"
      @input="updateTextLength"
      maxlength="3000" ></textarea>
      <p class="max-text-length"><span>{{ textLength }}</span> / 3000자</p>
    </div>

    <div>
      <button class="connect-btn btn" @click="connectQMS">Connect QMS</button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      qlikConenctConfig: {
        host: process.env.VUE_APP_QLIK_HOST,
        prefix: "/"+process.env.VUE_APP_QLIK_PROXY_PRIFIX+"/",
        port: 443,
        isSecure: true
      },
      isLoading: false,
      accessToken: '',

      status: '',
      issueSummary: '', 
      textLength: 0, 
      qUserId: '',
      qKeyValue: '',
      qSerialSn: '',

      appid: '',
      vUser: '',
      vKey: '',
      vContent: '',
      vSerial: '',

      qSeparator: '#qVar@',
    };
  },
  created(){
    const urlSearch = new URLSearchParams(location.search)
    var appid = urlSearch.get('appid');
    var qvar = urlSearch.get('qvar');

    if(appid){
      this.appid = appid;
    }else{
      this.appid = '88ff9953-979e-491a-b27c-602102388d66';
    }
    if(qvar){
      const arr = qvar.split(',');
      this.vUser = arr[0];
      this.vKey = arr[1];
      this.vContent = arr[2];
      this.vSerial = arr[3];
    }else{
      this.vUser = 'vUser';
      this.vKey = 'vKey';
      this.vContent = 'vContent';
      this.vSerial = 'vSerial';
    }
  },
  mounted() {
    this.initialize();
  },
  methods: {
    async initialize(){
      this.isLoading = true;
      try{
        await this.getToken();
        await this.fetchData();
        await this.loadExternalScript(process.env.VUE_APP_QLIK_URL 
        + 'resources/assets/external/requirejs/require.js');
        await this.afterScriptLoaded();
      }catch(error){
        console.error("Error during initialization:", error);
      }finally{
        this.isLoading = false;
      }
    },
    async getToken(){
      try{
        const response = await fetch(process.env.VUE_APP_API_URL + 'jwt/getToken', {
          method : 'GET'
        });
        if (!response.ok){
          throw new Error(`Server error: ${response.status}`);
        }
        const data = await response.json();
        console.log('Data received:', data.accessToken);
        this.accessToken = data.accessToken;
      }catch(error){
        alert(error.message);
      }
    },
    async fetchData() {
      try {
        const response = await fetch(process.env.VUE_APP_QLIK_URL 
        + 'qrs/about?xrfKey=1234567890abcdef',{
          method: 'GET',
          mode: 'cors',
          credentials: 'include',
          headers: {
              'Content-Type' : 'application/json',
              'X-Qlik-xrfKey' : '1234567890abcdef',
              'Authorization' : 'Bearer ' + this.accessToken,
          },
        });
        if (!response.ok) {
          throw new Error(`Server error: ${response.status}`);
        }
        const data = await response.json();
        console.log('Data received:', data);
      } catch (error) {
        this.error = error.message;
      }
    },
    async loadExternalScript(src) {
      return new Promise((resolve, reject) => {
        const script = document.createElement('script');
        script.src = src;
        script.onload = resolve;
        script.onerror = reject;
        document.head.appendChild(script);
      });
    },
    async afterScriptLoaded() {
      const require = window.require;

      // Qlik API의 require.js 설정
      require.config({
        baseUrl: (this.qlikConenctConfig.isSecure ? "https://" : "http://")
          + this.qlikConenctConfig.host
          + (this.qlikConenctConfig.port ? ":" + this.qlikConenctConfig.port : "")
          + this.qlikConenctConfig.prefix + "resources"
      });

      return new Promise((resolve, reject) => {
        // 외부 스크립트를 로드한 후 qlik 모듈 가져오기
        require(["js/qlik"], (qlik) => {
          qlik.on("error", (error) => {
            console.log("error :" + error.message);
            reject(error); // 에러 발생 시 reject 호출
          });

          console.log('appID ::: ' + this.appid);

          // Qlik App 연결
          const app = qlik.openApp(this.appid, this.qlikConenctConfig);

          // 첫 번째 변수 가져오기 및 평가
          app.variable.getContent(this.vUser, (reply) => {
            const qFormula = reply.qContent.qString;
            app.model.enigmaModel.evaluate(qFormula).then(value => {
              console.log("userID : " + value);
              this.qUserId = value;

              // 두 번째 변수 가져오기
              app.variable.getContent(this.vKey, (reply) => {
                const keyValue = reply.qContent.qString;
                console.log("key : " + keyValue);
                this.qKeyValue = keyValue;

                // 세 번째 변수 가져오기
                app.variable.getContent(this.vSerial, (reply) => {
                  const serialValue = reply.qContent.qString;
                  console.log("serial : " + serialValue);
                  this.qSerialSn = serialValue;

                  // 모든 작업이 완료되었을 때 resolve 호출
                  resolve();
                });
              });
            }).catch(err => {
              console.error("Error evaluating qUserId:", err);
              reject(err); // 에러 발생 시 reject 호출
            });
          });
        });
      });
    },
    save() {
      if(!this.status){
        alert('Status is null')
        return;
      }
      if(!this.issueSummary){
        alert('issueSummary is null')
        return;
      }
      const contentValue = 'i'+this.qSeparator+this.qKeyValue+this.qSeparator+this.issueSummary;
      //set Qlik Variable
      const require = window.require;
      require( ["js/qlik"], ( qlik ) => {
        const app = qlik.openApp(this.appid, this.qlikConenctConfig);
        app.variable.setStringValue(this.vContent, contentValue)
        .then(()=>{
          app.doReload(0, true).then(()=>{app.doSave()});
        }); 
      });

      const basisYm = this.getBasisYm();
      //Call Save API 
      try{
        fetch(process.env.VUE_APP_API_URL + "issueNote/save", {
          method: "POST",
          mode: "cors",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify({
            basisYm: basisYm,
            judgeBasisSn: this.qSerialSn,
            surrogateKeyCode: this.qKeyValue,
            progressStatusCode: this.status,
            issueSumDesc: this.issueSummary,
            userId: this.qUserId
          }),
        })
        .then((response) => response.json())
        .then((data) => console.log(data.detailMessage))
      }catch(error){
        alert(error.meesage)
      }

      alert('Selected Status : ' + this.status 
      + '\nInput Summary : ' 
      + this.issueSummary + '\n\n Saved!');
    },
    close() {
      window.close();
    },
    connectQMS() {
      this.getDetails();
    },
    updateTextLength() {
      this.textLength = this.issueSummary.length;
    },
    getDetails(){
      const basisYm = this.getBasisYm();
      try{
        fetch(process.env.VUE_APP_API_URL + "issueNote/detail", {
        method: "POST",
        mode: "cors",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          basisYm: basisYm,
          judgeBasisSn: this.qSerialSn,
          surrogateKeyCode: this.qKeyValue,
        }),
      })
      .then((response) => response.json())
      .then((data) => console.log(data))
      }catch(error){
        alert(error.message);
      }
    },
    getBasisYm(){
      var today = new Date();

      var year = today.getFullYear();
      var month = ('0' + (today.getMonth() + 1)).slice(-2);
      var basisYm = year + '-' + month;

      return basisYm;
    }
  }
};
</script>

<style scoped>
.spinner-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: rgba(0, 0, 0, 0.1);
  z-index: 1000;
}

.spinner {
  border: 16px solid #f3f3f3;
  border-top: 16px solid #3498db;
  border-radius: 50%;
  width: 120px;
  height: 120px;
  animation: spin 2s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

body {
    font-family: Arial, sans-serif;
    margin: 20px;
}

.top-buttons {
    display: flex;
    justify-content: flex-end;
    align-items: center;
    margin-bottom: 20px;
}

.container {
    max-width: 600px;
    margin: auto;
    border: 1px solid #ccc;
    padding: 20px;
    border-radius: 8px;
    background-color: #f9f9f9;
    max-height: 600px; /* 최대 높이 설정 */
}

.section-title {
    background-color: #f4b083;
    padding: 5px;
    margin-bottom: 10px;
    font-weight: bold;
}
.section-content {
    margin-bottom: 20px;
}

.inline-elements {
    display: flex;
    gap: 20px;
    align-items: center;
}

label {
    font-weight: bold;
}

select, textarea {
    padding: 8px 5px;
    border-radius: 4px;
    border: 1px solid #ccc;
}

select {
    width: 150px;
}

textarea {
    width: 97%;
    height: 150px;
    resize: none;
}

.max-text-length{
    margin: 5px 5px;
    font-size: 12px;
    text-align: right;
}
.table-container {
    max-height: 250px; /* 테이블 컨테이너의 최대 높이 설정 */
    overflow-y: auto; /* 테이블 컨테이너에 스크롤바 추가 */
}

table {
    width: 100%;
    border-collapse: collapse;
    margin-bottom: 20px;
}

table, th, td {
    border: 1px solid #ccc;
    text-align: center;
}

th, td {
    padding: 10px;
}

.btn{
    background-color: #4CAF50;
    color: white;
    padding: 10px 20px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    margin-right: 10px;
    box-shadow: 0 4px 8px rgba(0,0,0,0.20);
}

.btn:hover{
    opacity: 0.8;
}

.close-btn {
    background-color: #f44336;
    margin-right: 0;
}

.connect-btn{
    background-color: #f0773c;
    display: flex;
}
</style>
