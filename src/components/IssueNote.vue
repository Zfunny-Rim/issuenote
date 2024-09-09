<template>
  <div class="container">
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
      <textarea id="issue-summary" v-model="issueSummary" placeholder="내용작성 (이슈칼럼임)" @input="updateTextLength"></textarea>
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

      status: '',
      issueSummary: '', 
      textLength: 0, 
      qUserId: '',
      qKeyValue: '',

      appid: '',
      vUser: '',
      vKey: '',
      vContent: '',
    };
  },
  mounted() {
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
    }else{
      this.vUser = 'vUser';
      this.vKey = 'vKey';
      this.vContent = 'vContent';
    }
    this.getToken();
  },
  methods: {
    save() {
      if(!this.status){
        alert('Status is null')
        return;
      }
      if(!this.issueSummary){
        alert('issueSummary is null')
        return;
      }
      //set Qlik Variable
      const require = window.require;
      require( ["js/qlik"], ( qlik ) => {
        const app = qlik.openApp(this.appid, this.qlikConenctConfig);
        app.variable.setStringValue(this.vContent, this.issueSummary); 
      });
      
      //Call Save API
      try{
        fetch(process.env.VUE_APP_API_URL + "issueNote/save", {
          method: "POST",
          mode: "cors",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify({
            basisYm: "2024-08",
            judgeBasisSn: "Test2",
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
        this.fetchData(data.accessToken);
      }catch(error){
        alert(error.message);
      }
    },
    async fetchData(token) {
      try {
        const response = await fetch(process.env.VUE_APP_QLIK_URL 
        + 'qrs/about?xrfKey=1234567890abcdef',{
          method: 'GET',
          mode: 'cors',
          credentials: 'include',
          headers: {
              'Content-Type' : 'application/json',
              'X-Qlik-xrfKey' : '1234567890abcdef',
              'Authorization' : 'Bearer ' + token,
          },
        });
        if (!response.ok) {
          throw new Error(`Server error: ${response.status}`);
        }
        const data = await response.json();
        console.log('Data received:', data);

        await this.loadExternalScript(process.env.VUE_APP_QLIK_URL 
        + 'resources/assets/external/requirejs/require.js');

        this.afterScriptLoaded();
      } catch (error) {
        this.error = error.message;
      }
    },
    loadExternalScript(src) {
      return new Promise((resolve, reject) => {
        const script = document.createElement('script');
        script.src = src;
        script.onload = resolve;
        script.onerror = reject;
        document.head.appendChild(script);
      });
    },
    afterScriptLoaded() {
      const require = window.require;
      require.config( {
        baseUrl: ( this.qlikConenctConfig.isSecure ? "https://" : "http://" ) 
        + this.qlikConenctConfig.host 
        + (this.qlikConenctConfig.port ? ":" + this.qlikConenctConfig.port : "") 
        + this.qlikConenctConfig.prefix + "resources"
      } )
      require( ["js/qlik"], ( qlik ) => {
        qlik.on( "error",( error ) => {
          console.log("error :" + error.message);
        });

        console.log('appID ::: ' + this.appid);

        var app = qlik.openApp(this.appid, this.qlikConenctConfig);
        app.variable.getContent(this.vUser, ( reply ) => {
          var qFormula = reply.qContent.qString;
          app.model.enigmaModel.evaluate(qFormula).then(value => {
              console.log("userID : " + value);
              this.qUserId = value;
          });
        });

        app.variable.getContent(this.vKey, ( reply ) => {
          var value = reply.qContent.qString;
          console.log("key : " + value);
          this.qKeyValue = value;
        });
      });
    },
    getDetails(){
      try{
        fetch(process.env.VUE_APP_API_URL + "issueNote/detail", {
        method: "POST",
        mode: "cors",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          basisYm: "Test1",
          judgeBasisSn: "Test2",
          surrogateKeyCode: "Test3"
        }),
      })
      .then((response) => response.json())
      .then((data) => console.log(data))
      }catch(error){
        alert(error.message);
      }
    }
  }
};
</script>

<style scoped>
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
