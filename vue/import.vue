
<template>
  <div class="importPage">
    <p class="importIntro">Here you can import configuration from a template. Both OpenBeken templates and Cloudcutter profiles are supported. Applying the import will overwrite your current pin/channel assignments and startup command, and it will clear LittleFS (including <code>autoexec.bat</code>).</p>
    <div class="importContainer">
      <div class="importItem"  style="width: 300px;">
        <h3>1. Paste a template or profile</h3>
        <p>Paste an <a href="https://openbekeniot.github.io/webapp/devicesList.html">OpenBeken template</a> or Cloudcutter JSON. You can find Cloudcutter device profiles <a href="https://github.com/tuya-cloudcutter/tuya-cloudcutter.github.io/tree/master/devices">here</a>.</p>
        <textarea id="importTemplate" 
        placeholder="Paste (or drag and drop) an OpenBeken template, Tuya JSON, or Cloudcutter JSON here" style="vertical-align: top; width: 280px; height:500px" @input="handleImportTemplateChange" v-model="importTemplateText"></textarea>
       <br/>
               <br>
              <div class="importExamplesTitle">Developer testing examples:</div>
       <div class="importExamples">
         <button @click="loadDemo('https://raw.githubusercontent.com/tuya-cloudcutter/tuya-cloudcutter.github.io/a8a6539aad21a03d5db41e4d27e9d5516c62fe23/devices/lenovo-se-242dc-rgbct-bulb-v1.2.21.json')">RGBCW LED</button>
         <button @click="loadDemo('https://raw.githubusercontent.com/tuya-cloudcutter/tuya-cloudcutter.github.io/a8a6539aad21a03d5db41e4d27e9d5516c62fe23/devices/nous-p4-e14-rgbct-bulb.json')">BP5758 LED</button>
         <button @click="loadDemo('https://raw.githubusercontent.com/tuya-cloudcutter/tuya-cloudcutter.github.io/b04e860fe0bb1c8bed417ab36c57e6759ec08510/devices/spectrum-woj14415-rgbct-gu10-bulb.json')">SM2135 LED</button>
          <button @click="loadDemo('https://raw.githubusercontent.com/tuya-cloudcutter/tuya-cloudcutter.github.io/a8a6539aad21a03d5db41e4d27e9d5516c62fe23/devices/pegant-pg3451-3-outlet-power-strip.json')">Triple socket + USB</button>
          <button @click="loadDemo('https://raw.githubusercontent.com/tuya-cloudcutter/tuya-cloudcutter.github.io/b04e860fe0bb1c8bed417ab36c57e6759ec08510/devices/tuya-generic-rr620w-jl-smart-switch.json')">BL0942 plug</button>
          <button @click="loadDemo('https://raw.githubusercontent.com/tuya-cloudcutter/tuya-cloudcutter.github.io/43a341ef1dd8eec8514e1d435563bd9008ff2835/devices/hombli-hbss-0209-smart-socket-b2030248-energy-plug.json')">BL0937 plug</button>
          <button @click="loadDemo('https://raw.githubusercontent.com/tuya-cloudcutter/tuya-cloudcutter.github.io/a8a6539aad21a03d5db41e4d27e9d5516c62fe23/devices/athom-garage-door-opener.json')">Garage switch</button>
          <button @click="loadDemo('https://raw.githubusercontent.com/tuya-cloudcutter/tuya-cloudcutter.github.io/a8a6539aad21a03d5db41e4d27e9d5516c62fe23/devices/aldi-casalux-wifi-led-rgb-light-strip.json')">RGB LED</button>
       </div>
</div>

      <div class="importItem"  style="width: 300px;">
        <h3>2. Review the generated script (edit if needed)</h3>
               <br>
        <p id ="generateTextID" v-text="generateText"></p>
        <textarea id="generatedScriptField" placeholder="The generated script will appear here" style="vertical-align: top; width: 280px; height:500px"  v-model="generatedScriptText"></textarea>
       <br>
      </div>

      <div class="importItem" style="width: 300px;">
      <h3> 3. Apply the script to the device</h3>
               <br>
       <button @click="applyScript()">Apply script (clears current configuration)</button>
       <br>
       <br>
       <div class="importExecTitle">Execution log:</div>
        <p id ="progressTextID" v-text="progressText"> </p>
      </div>
    </div>
  </div>
</template>



<script>
  module.exports = {
    components: {
      'import': window.getComponent('import')
    },
    data: ()=>{
      return {
        generateText:"",
        importTemplateText: "",
        generatedScriptText: "",
        progressText: "",
        generateTextElem:undefined,
        progressTextElem:undefined,
      }
    },
    methods:{
      getinfo(){
       

      },
      onSendFailed(response,line) {
        //alert('Send failed!');
        if(response.status == 501 || response.status == 400) {
           this.progressTextElem.innerHTML += "<span style='color:red;'>Failed: invalid command \""+line+"\".</span>";
        } else {
           this.progressTextElem.innerHTML += "<span style='color:red;'>Failed. Check your network connection and try again.</span>";
        }
      },
      async sendLines(lines) {
        this.progressTextElem.innerHTML = "Sending...";
        let idx = 0;
      for (let line of lines) {
        let dbg = "";
        if(false){
          dbg = " ("+line+")";
        }
        this.progressTextElem.innerHTML += " Sending "+idx+" of "+lines.length+dbg+"...";
        await this.sendLine(line,this.onSendFailed);
        idx++;
      }
        this.progressTextElem.innerHTML += "<span style='color:green;'> Completed.</span>";
        this.progressTextElem.innerHTML += "<span style='color:green;'> Restart the device if required.</span>";
    },
    async sendLine(line, errorHandler) {
      line = line.trim();
        if(line.length < 2)
          return;
        if(line[0]=='/' && line[1] == '/')
            return;
      console.log("sending line: " + line);
      let url = window.device+'/api/cmnd';
      return new Promise((resolve, reject) => {
        // Replace the URL and request options with your own
        fetch(url, {
          method: "POST",
          body: line 
        })
        .then(response => {
          if (!response.ok) {
           errorHandler(response,line);
            throw new Error("Failed to send line "+line + " with error " + response.status);
          }
          resolve();
        })
        .catch(error => {
          reject(error);
        });
      });
    },
      applyScript() {

        const lines = this.generatedScriptText.split("\n");

       // lines.unshift("ClearIO");
        for (let i = 0; i < lines.length; i++) {
          const line = lines[i];
         // alert("line " +line);
        }
        this.sendLines(lines);
      },
      setImportSrc(txt){
        //alert(txt);
        this.importTemplateText  = txt;
       this.refreshTemplateImport();
      },
      loadDemo(url) {
            fetch(url)
                .then(response => response.text())
                .then(res => {
                  this.setImportSrc(res);
                })
                .catch(err => {
                    this.error = err.toString();
                    console.error(err)
                }); // Never forget the final catch!
      },
    handleImportTemplateChange(event) {
      console.log("Import template changed!");
       this.refreshTemplateImport();
    },
    refreshTemplateImport() {
       let res;
       if(processJSON==undefined){
           this.generateTextElem.innerHTML = "<span style='color:red;'>Template parser is not loaded yet.</span>";
          return;
       }
       let jsonText = this.importTemplateText;
       jsonText = jsonText.trim();
       if(jsonText.length < 1) {
           this.generateTextElem.innerHTML = "<span style='color:yellow;'>No input provided.</span>";
          return;
       }
       try {
           res = processJSON(jsonText);
       } catch (error) {
           this.generateTextElem.innerHTML = "<span style='color:red;'>Failed: "+error+".</span>";
           return;
       }
        this.generatedScriptText  = "";
         this.generatedScriptText  +=  "ClearIO // clear old GPIO/channels"+"\n";
         this.generatedScriptText  +=  "lfs_format // clear LittleFS"+"\n";
         this.generatedScriptText  +=  "StartupCommand \"\"  // clear STARTUP"+"\n";
         this.generatedScriptText  +=  "stopDriver *  // stop drivers"+"\n";
         this.generatedScriptText  += res.scr;
           this.generateTextElem.innerHTML = "<span style='color:green;'>OK. Script generated.</span>";
        
    },
      
      handleDragOver(event) {
        event.preventDefault();
      },
      handleDrop(event) {
        event.preventDefault();

        const files = event.dataTransfer.files;

        if (files.length > 0) {
          const reader = new FileReader();

          reader.onload = (e) => {
            this.importTemplateText = e.target.result;
          };

          reader.readAsText(files[0]);
        }
      },

    },
    mounted (){
        this.msg = 'fred';
        this.progressTextElem = document.getElementById("progressTextID");
        this.generateTextElem = document.getElementById("generateTextID");
        this.generateTextElem.innerHTML = "Waiting for input.";
        const plugin = document.createElement("script");
        plugin.setAttribute(
          "src",
          "https://openbekeniot.github.io/webapp/templateParser.js"
         //"../templateParser.js"
        );
        plugin.async = true;
        document.head.appendChild(plugin);
      let importTemplateTextarea = document.getElementById("importTemplate");
      importTemplateTextarea.addEventListener('dragover', this.handleDragOver);
      importTemplateTextarea.addEventListener('drop', this.handleDrop);
    },
    destroyed(){
      clearInterval(this.interval);
    }

  }
//@ sourceURL=/vue/import.vue
</script>

<style scoped>
  .importPage {
    width: 100%;
  }

  .importIntro {
    max-width: 940px;
    margin: 0 auto 36px auto;
    padding: 0 15px;
  }

  .importContainer {
    display: flex;
    justify-content: center;
    align-items: flex-start;
    flex-wrap: wrap;
    padding-top: 6px;
  }

  .importItem {
    padding: 0 15px;
  }

  .importExamplesTitle {
    margin: 0 0 6px 0;
  }

  .importExamples {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  .importExecTitle {
    margin-bottom: 6px;
  }

  .pin-index {
    display: inline-block;
    width: 20px;
  }
</style>

