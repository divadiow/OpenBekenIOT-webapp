<template>
    <div>
        <div>
            <div class="flashIntro">
            <p>Use this page to read and download flash regions directly from the device, and write RF/CFG partitions. The RF partition contains Beken calibration and network identity data, while the OBK configuration contains OpenBeken settings.</p>
            <p>If your device has an invalid MAC address ending in <code>00:00:00</code>, use Restore RF configuration (supported on BK7231N/T2/T34, BK7231T, BK7231U, BK7252, BK7231M, BK7258/T5, BK7238/T1, BK7252N/T4, BK7236/T3). This rewrites the RF partition from a built-in template and generates a new MAC address by randomising the last 3 bytes of the MAC address. Reboot the device afterward.</p>
            <p><b>BK7238/T1 note:</b> If you have a backup of the device's factory RF partition (TLV data), you can restore it using <i>Write RF data to device</i>. OpenBK7238 stores RF at <code>0x1E0000</code>, while stock Tuya BK7238 firmware typically stores it at <code>0x1E3000</code>. Writing a known-good backup restores the factory MAC address and may also restore RF calibration data.</p>
            </div>


            <table class="my-table">
            <tr>
                <th>Read</th>
                <th>Download</th>
                <th>Write</th>
                <th>Extra</th>
            </tr>
            <tr>
                <td> <button @click="rf(null, $event)">Read RF configuration</button></td>
                <td><a :href="rfurl" download="rfdata">Download RF data</a></td>
                <td>
                    <div class="rfWriteBlock">
                        <div class="rfWriteLabel">
                            <label for="RFFilePicker">Select a file with RF TLV data (TLV header), or a full flash dump containing RF TLV:</label>
                        </div>
                        <div class="rfWriteFile">
                            <input id="RFFilePicker" type="file" @change="RFFileSelected($event)">
                        </div>
                        <div class="rfWriteButton">
                            <button @click="writeRFCFG(null, $event)" :disabled="!isWriteRFSupported || invalidRFCFGSelected || !rfCfgData" :title="writeRFTitle">{{ writeRFLabel }}</button>
                        </div>
                    </div>
                </td>
                <td><button @click="restore_rf(null, $event)" :disabled="!isRestoreRFSupported" :title="restoreRFTitle">{{ restoreRFLabel }}</button></td>
            </tr>
            <tr>
                <td>  <button @click="config(null, $event)">Read OBK configuration</button></td>
                <td> <a :href="configurl" download="configdata">Download OBK configuration</a></td>
                <td>
                <div class="cfgWriteBlock">
                    <div class="cfgWriteLabel">
                        <label for="cfgFilePicker">Select a file with a binary CFG header:</label>
                    </div>
                    <div class="cfgWriteFile">
                        <input id="cfgFilePicker" type="file" @change="cfgFileSelected($event)">
                    </div>
                    <div class="cfgWriteButton">
                        <button @click="writeCFG(null, $event)">Write CFG to device</button>
                    </div>
                </div>
                </td>
                <td><button @click="downloadFullDump(null, $event)">Download full 2MB flash dump</button></td>
            </tr>
            <tr>
                <td>  <button @click="readTuyaConfig(null, $event)">Read Tuya GPIO configuration</button></td>
                <td> <button @click="downloadTuyaConfig(null, $event)">Download Tuya GPIO configuration</button></td>
                <td></td>
                <td></td>
            </tr>
            <tr>
                <td>  <button @click="readCustom(null, $event)">Read custom region (prompts for offset/length)</button></td>
                <td> <button @click="downloadCustom(null, $event)">Download custom region (prompts for offset/length)</button></td>
                <td></td>
                <td></td>
            </tr>
            </table>
                <div class="jobStatus">
                    <h4>Job Status</h4>
                    <div class="jobStatusLog" v-html="status"></div>
                </div>
</div>
        <div v-html="display" class="display"></div>
    </div>
</template>

<script>
const RF_START_BY_CHIPSET = {
    BK7231T: '1e0000',
    BK7231U: '1e0000',
    BK7252:  '1e0000',
    BK7238:  '1e0000',
    BK7252N: '1e0000',
    BK7231N: '1d0000',
    BK7231M: '1d0000',
    BK7236:  '3fe000',
    BK7258:  '7fe000',
};

  module.exports = {

    data: ()=>{
      return {
        rfdata: null,
        fullDumpData: null,
        fullDumpCurAt: 0,
        fullDumpFlashStart: 0,
        fullDumpFlashSize: 0,
        fullDumpErrors: 0,
        fullDumpRunning:0,
        fullDumpChunkSize: 0,
        fullDumpJobLabel: '',
        fullDumpStyle: '',
        display: '',
        configdata: null,
        cfgdata: null,
        build:'unknown',
        chipset:'unknown',
        status:'',
        error: '',
        shortName:'',
        rfurl: '',
        configurl: '',
        invalidCFGSelected: true,
        rfCfgData: null,
        invalidRFCFGSelected: true,
        rfCfgSourceName: '',
        rfCfgSourceOffset: null,
        rfCfgSourceWasExtracted: false,
      }
    },
    computed:{
        isRestoreRFSupported(){
            // Keep this list in sync with restore_rf_internal() templates.
            return this.chipset === 'BK7231T' || this.chipset === 'BK7231U' || this.chipset === 'BK7252'
                || this.chipset === 'BK7231N' || this.chipset === 'BK7231M' || this.chipset === 'BK7258'
                || this.chipset === 'BK7238' || this.chipset === 'BK7252N'
                || this.chipset === 'BK7236';
        },
        restoreRFLabel(){
            if(!this.chipset || this.chipset === 'unknown'){
                return 'Restore RF configuration (detecting chipset...)';
            }
            if(this.isRestoreRFSupported){
                return 'Restore RF configuration (' + this.chipset + ')';
            }
            return 'Restore RF configuration (unsupported: ' + this.chipset + ')';
        },
        restoreRFTitle(){
            if(!this.chipset || this.chipset === 'unknown'){
                return 'Waiting for /api/info to determine chipset.';
            }
            if(this.isRestoreRFSupported){
                return 'Writes generic RF calibration + randomizes MAC (last 3 bytes). Reboot afterward.';
            }
            return 'Restore RF is supported only on BK7231T/U, BK7231N/M, BK7236, BK7238, BK7252, BK7252N, BK7258.';
        },
        isWriteRFSupported(){
            // Keep write-RF enablement aligned with Restore RF support + getRFAddress() mapping.
            return this.isRestoreRFSupported;
        },
        writeRFLabel(){
            if(!this.chipset || this.chipset === 'unknown'){
                return 'Write RF data to device (detecting chipset...)';
            }
            if(this.isWriteRFSupported){
                const parts = this.getRFAddress().split('-');
                const startHex = (parts[0] || '').toUpperCase();
                return 'Write RF data to device (' + this.chipset + ' @ 0x' + startHex + ')';
            }
            return 'Write RF data to device (unsupported: ' + this.chipset + ')';
        },
        writeRFTitle(){
            if(!this.chipset || this.chipset === 'unknown'){
                return 'Waiting for /api/info to determine chipset.';
            }
            if(this.isWriteRFSupported){
                const parts = this.getRFAddress().split('-');
                const startHex = (parts[0] || '').toUpperCase();
                const lenHex = (parts[1] || '').toUpperCase();
                return 'Writes RF partition at 0x' + startHex + ' (len 0x' + lenHex + ') using the selected file. Accepts RF TLV (TLV\\0 at offset 0) or a dump containing RF TLV. File size limit: 8MB.';
            }
            return 'Write RF is supported only on BK7231T/U, BK7231N/M, BK7236, BK7238, BK7252, BK7252N, BK7258.';
        },
    },
    methods:{

        escapeHtml(s){
            if(s === null || s === undefined) return '';
            return ('' + s)
                .replace(/&/g, '&amp;')
                .replace(/</g, '&lt;')
                .replace(/>/g, '&gt;')
                .replace(/"/g, '&quot;')
                .replace(/'/g, '&#39;');
        },
        buffersEqual(a, b){
            if(!a || !b) return false;
            if(a.byteLength !== b.byteLength) return false;
            const va = new Uint8Array(a);
            const vb = new Uint8Array(b);
            for(let i=0;i<va.length;i++){
                if(va[i] !== vb[i]) return false;
            }
            return true;
        },
        macPreviewFromRFBlock(arrayBuffer){
            try{
                if(!arrayBuffer || arrayBuffer.byteLength < (36 + 6)){
                    return '(n/a)';
                }
                const view = new Uint8Array(arrayBuffer, 36, 6);
                let parts = [];
                for(let i=0;i<6;i++){
                    parts.push(('0' + view[i].toString(16)).slice(-2).toUpperCase());
                }
                return parts.join(':');
            }catch(e){
                return '(n/a)';
            }
        },


        getinfo(){
            let url = window.device+'/api/info';
            fetch(url)
                .then(response => response.json())
                .then(res => {
                    this.build = res.build;
                    this.shortName = res.shortName;
                    this.chipset = res.chipset;     //Set chipset to fixed value for testing
                    this.rfurl = window.device+'/api/flash/'+this.getRFAddress();
                    this.configurl = window.device+'/api/flash/'+this.getConfigAddress();
                })
                .catch(err => {
                    this.error = err.toString();
                    this.status += '<br/>Error fetching /api/info: ' + this.escapeHtml(this.error);
                    console.error(err);
                }); // Never forget the final catch!
        },
        writeCFG(cb){
            if(this.invalidCFGSelected){
                alert("Invalid CFG file selected.");
                return;
            }

            let confirmationForUser = prompt("Are you sure? This will overwrite your OBK configuration (pins, channels, flags, WiFi, IP, MQTT, short startup command). Type Y to continue.", "N");
            if(confirmationForUser != null && confirmationForUser.length > 0 && (confirmationForUser[0] === 'Y' || confirmationForUser[0] === 'y')) {
                this.status += '<br/>Starting CFG write...';
                this.writeCFG_Internal(cb);
            }
        },
        writeCFG_Internal(cb){
            let url = window.device+'/api/flash/'+this.getConfigAddress();
            console.log('Will use URL '+url);

            const t0 = performance.now();
            fetch(url, {
                    method: 'POST',
                    body: this.cfgdata
                })
                .then(async response => {
                    const text = await response.text();
                    const dt = Math.round(performance.now() - t0);

                    this.status += '<br/>CFG write: POST returned HTTP ' + response.status + ' in ' + dt + 'ms.';
                    if(text && text.length){
                        this.status += '<br/>CFG write: device response: ' + this.escapeHtml(text);
                    }
                    if(!response.ok){
                        throw new Error('HTTP ' + response.status);
                    }

                    this.status += '<br/>CFG write: complete. Reboot the device to apply changes.';
                    if(cb) cb();
                })
                .catch(err => {
                    console.error(err);
                    this.status += '<br/>CFG write: failed: ' + this.escapeHtml(err.toString());
                }); // Never forget the final catch!
        },
        cfgFileSelected(ev){
            console.log("CFG File selected");
            this.invalidCFGSelected = true; // pessimistic until validated
            this.cfgdata = null;

            var file = ev.target.files[0];  //There should be only one file
            if (file){
                this.status += '<br/>CFG file selected: <b>' + this.escapeHtml(file.name) + '</b> (' + file.size + ' bytes).';
                console.log('... CFG fileName = ' + file.name);
                var reader = new FileReader();
                reader.onload = (event) => this.checkCFGData(event, file, "selected");
                reader.readAsArrayBuffer(file);
            }
        },
        isCFGImage(arrayBuffer){
            let view = new DataView(arrayBuffer);
            if (view.byteLength < 3) return false;
            console.log("isCFGImage sees " +view);
            // CFG
            return view.getUint8(0) === 0x43 && view.getUint8(1) === 0x46 && view.getUint8(2) === 0x47;
        },
        checkCFGData(event, file, operation){
            this.cfgdata = null;

            var result = event.target.result;   //ArrayBuffer
            console.log('chipset=' + this.chipset);
            console.log("Checking CFG data");
            console.log('cfgdata len:' + result.byteLength);

            this.invalidCFGSelected = !this.isCFGImage(result);

            if (this.invalidCFGSelected){
                console.log("this cfg is incorrect");
                this.status += '<br/>CFG file rejected: missing CFG header.';
            }
            else{
                console.log("this cfg is OK");
                this.cfgdata = result;
                this.status += '<br/>CFG file accepted: valid CFG header.';
            }
        },

                // Write RF partition
        writeRFCFG(cb){
            if(!this.isWriteRFSupported){
                alert("Write RF is unsupported on chipset: " + this.chipset);
                return;
            }
            if(this.invalidRFCFGSelected || !this.rfCfgData){
                alert("Invalid RF data file selected.");
                return;
            }

            let confirmationForUser = prompt("Are you sure? This will overwrite your RF configuration (MAC address and calibration). Type Y to continue.", "N");
            if(confirmationForUser != null && confirmationForUser.length > 0 && (confirmationForUser[0] === 'Y' || confirmationForUser[0] === 'y')) {
                this.status += '<br/>Starting RF write...';
                this.writeRFCFG_Internal(cb);
            }
        },
        writeRFCFG_Internal(cb){
            const rfAdr = this.getRFAddress();
            const parts = (rfAdr || '').split('-');
            const startHex = ((parts[0] || '') + '').toUpperCase();
            const lenHex = ((parts[1] || '') + '').toUpperCase();

            let payload = this.rfCfgData;
            if(payload && payload.byteLength > 0x1000){
                this.status += '<br/>RF write: note - RF buffer is larger than <code>0x1000</code>; using first <code>0x1000</code> bytes.';
                payload = payload.slice(0, 0x1000);
            }

            const srcName = this.escapeHtml(this.rfCfgSourceName || 'selected file');
            const srcOff = (this.rfCfgSourceOffset === null || this.rfCfgSourceOffset === undefined) ? null : this.rfCfgSourceOffset;
            const srcDesc = this.rfCfgSourceWasExtracted && srcOff !== null
                ? ' (extracted, TLV @ <code>0x' + srcOff.toString(16).toUpperCase() + '</code>)'
                : ' (direct TLV)';
            const macPrev = this.macPreviewFromRFBlock(payload);

            this.status += '<br/>RF write: source <b>' + srcName + '</b>' + srcDesc + ', posting ' + payload.byteLength + ' bytes to <code>0x' + startHex + '</code> (len <code>0x' + lenHex + '</code>). MAC preview (offset 36): ' + macPrev + '.';

            let url = window.device+'/api/flash/'+rfAdr;
            console.log('Will use URL '+url);

            const t0 = performance.now();
            fetch(url, {
                method: 'POST',
                body: payload
            })
            .then(async response => {
                const text = await response.text();
                const dt = Math.round(performance.now() - t0);

                this.status += '<br/>RF write: POST returned HTTP ' + response.status + ' in ' + dt + 'ms.';
                if(text && text.length){
                    this.status += '<br/>RF write: device response: ' + this.escapeHtml(text);
                }
                if(!response.ok){
                    throw new Error('HTTP ' + response.status);
                }
                return fetch(url);
            })
            .then(response => response.arrayBuffer())
            .then(buffer => {
                const ok = this.buffersEqual(payload, buffer);
                const readMac = this.macPreviewFromRFBlock(buffer);

                if(ok){
                    this.status += '<br/>RF write verify: <b>OK</b> (read-back matches). Read-back MAC preview (offset 36): ' + readMac + '.';
                } else {
                    this.status += '<br/>RF write verify: <b>FAILED</b> (read-back mismatch). Read-back MAC preview (offset 36): ' + readMac + '.';
                }

                if(cb) cb();
            })
            .catch(err => {
                console.error(err);
                this.status += '<br/>RF write: failed: ' + this.escapeHtml(err.toString());
            }); // Never forget the final catch!
        },
        RFFileSelected(ev){
            console.log("RF file selected");
            this.invalidRFCFGSelected = true; // pessimistic until validated
            this.rfCfgData = null;
            this.rfCfgSourceName = '';
            this.rfCfgSourceOffset = null;
            this.rfCfgSourceWasExtracted = false;

            var file = ev.target.files[0]; //There should be only one file
            if (file){
                this.rfCfgSourceName = file.name;

                this.status += '<br/>RF file selected: <b>' + this.escapeHtml(file.name) + '</b> (' + file.size + ' bytes).';

                const maxBytes = 8 * 1024 * 1024;
                if (file.size > maxBytes){
                    this.invalidRFCFGSelected = true;
                    this.rfCfgData = null;
                    this.status += '<br/>RF file rejected: file larger than 8MB. Please choose a smaller dump.';
                    return;
                }

                var reader = new FileReader();
                reader.onload = (event) => this.checkRFCFGData(event, file, "selected");
                reader.readAsArrayBuffer(file);
            }
        },
        isRFCFGImage(arrayBuffer){
            let view = new DataView(arrayBuffer);
            if (view.byteLength < 4) return false;
            // TLV\0 header
            return view.getUint8(0) === 0x54 && view.getUint8(1) === 0x4C && view.getUint8(2) === 0x56 && view.getUint8(3) === 0;
        },
        findTLVHeaderOffset(arrayBuffer){
            let view = new DataView(arrayBuffer);
            let len = view.byteLength;
            if (len < 4) return -1;

            const isTLVAt = (off) => {
                return view.getUint8(off) === 0x54 && view.getUint8(off+1) === 0x4C && view.getUint8(off+2) === 0x56 && view.getUint8(off+3) === 0;
            };

            // Try likely offsets first (based on detected chipset).
            let likely = [];
            try{
                let rf = this.getRFAddress();
                let parts = (rf || '').split('-');
                if(parts.length > 0){
                    let start = parseInt(parts[0], 16);
                    if(Number.isFinite(start)) likely.push(start);
                }
            }catch(e){
            }
            if(this.chipset === 'BK7238'){
                // Stock Tuya BK7238 firmwares often place RF at 0x1E3000.
                likely.push(0x1E3000);
            }

            let seen = {};
            for(let i=0;i<likely.length;i++){
                let off = likely[i];
                if(off === undefined || off === null) continue;
                if(seen[off]) continue;
                seen[off] = true;
                if(off >= 0 && off + 4 <= len){
                    if(isTLVAt(off)) return off;
                }
            }

            // Scan at 0x1000 alignment (RF partitions are typically 4KB).
            for(let off=0; off + 4 <= len; off += 0x1000){
                if(isTLVAt(off)) return off;
            }
            return -1;
        },
        checkRFCFGData(event, file, operation){
            this.rfCfgData = null;

            var result = event.target.result; //ArrayBuffer
            console.log('chipset=' + this.chipset);
            console.log("Checking RF data");
            console.log('rfdata len:' + result.byteLength);

            // 1) Accept raw RF files that start with TLV\0 at offset 0.
            if(this.isRFCFGImage(result)){
                this.invalidRFCFGSelected = false;
                this.rfCfgSourceOffset = 0;
                this.rfCfgSourceWasExtracted = false;

                // Use first 0x1000 bytes (RF partition size).
                let block = result;
                if(block.byteLength > 0x1000){
                    block = block.slice(0, 0x1000);
                }
                this.rfCfgData = block;

                const macPrev = this.macPreviewFromRFBlock(block);
                this.status += '<br/>RF TLV detected at <code>0x0</code>; using <code>0x1000</code> bytes. MAC preview (offset 36): ' + macPrev + '.';
                return;
            }

            // 2) If not TLV-at-0, scan the file for a TLV header and extract a 0x1000 RF block.
            let off = this.findTLVHeaderOffset(result);
            if(off >= 0 && (off + 0x1000) <= result.byteLength){
                let extracted = result.slice(off, off + 0x1000);
                if(this.isRFCFGImage(extracted)){
                    this.invalidRFCFGSelected = false;
                    this.rfCfgSourceOffset = off;
                    this.rfCfgSourceWasExtracted = true;
                    this.rfCfgData = extracted;

                    const macPrev = this.macPreviewFromRFBlock(extracted);
                    this.status += '<br/>RF TLV found inside file at <code>0x' + off.toString(16).toUpperCase() + '</code>; extracted <code>0x1000</code> bytes. MAC preview (offset 36): ' + macPrev + '.';
                    return;
                }
            }

            this.invalidRFCFGSelected = true;
            this.rfCfgData = null;
            this.rfCfgSourceOffset = null;
            this.rfCfgSourceWasExtracted = false;
            this.status += '<br/>RF file rejected: no TLV header found at offset 0 or at expected dump offsets.';
        },

getRFAddress(){
            // RF partition (usually 0x1000 bytes) address varies by chipset.
            const start = RF_START_BY_CHIPSET[this.chipset];
            if(start){
                return start + '-1000';
            }
			console.log('getRFAddress is not implemented for ' + this.chipset);
			return '1e0000-1000';
        },
        getConfigAddress(){
            // OBK config partition is typically RF+0x1000 (size 0x1000).
            const rfStart = RF_START_BY_CHIPSET[this.chipset];
            if(rfStart){
                const cfgStart = (parseInt(rfStart, 16) + 0x1000).toString(16);
                return cfgStart + '-1000';
            }
			console.log('getConfigAddress is not implemented for ' + this.chipset);
			return '1e1000-1000';
        },
        appendArrayBuffers(buffer1, buffer2) {
            const combinedLength = buffer1.byteLength + buffer2.byteLength;
            const combinedBuffer = new ArrayBuffer(combinedLength);
            const combinedView = new Uint8Array(combinedBuffer);

            const buffer1View = new Uint8Array(buffer1);
            const buffer2View = new Uint8Array(buffer2);

            combinedView.set(buffer1View, 0);
            combinedView.set(buffer2View, buffer1.byteLength);

            return combinedBuffer;
        },
        downloadArrayBuffer(arrayBuffer, filename) {
            const blob = new Blob([arrayBuffer]);
            const link = document.createElement('a');
            link.href = window.URL.createObjectURL(blob);
            link.download = filename;

            // Programmatically click the link to start the download
            link.click();

            // Clean up resources
            window.URL.revokeObjectURL(link.href);
        },
        generateFullDumpDownloadForBrowser() {
            let fileName = this.chipset + "_"+this.fullDumpStyle+"_";
            if(this.shortName!=undefined && this.shortName.length>0){
                fileName += this.shortName;
            }
            else{
                fileName += "Unnamed";
            }
            this.downloadArrayBuffer(this.fullDumpData, fileName+".bin");
        },

        startDumpJob(label, fileTag, flashStart, flashSize, style){
            this.fullDumpJobLabel = label;
            this.fullDumpFlashStart = flashStart;
            this.fullDumpFlashSize = flashSize;
            this.fullDumpStyle = style;
            this.doFlashDumpInternal();
        },
        getDumpJobLabel(){
            if(this.fullDumpJobLabel!=undefined && this.fullDumpJobLabel.length>0){
                return this.fullDumpJobLabel;
            }
            return "full dump";
        },
        onFullDumpReadyForDownload(){
            this.fullDumpRunning = 0;
            this.status += `<br/>${this.getDumpJobLabel()} is ready.`;
            this.generateFullDumpDownloadForBrowser();
        },
        downloadFullDumpFragment(){
            // set address and size to string like "1e3000-2000"
            let fullAdr = this.fullDumpCurAt.toString(16) + "-"+ this.fullDumpChunkSize.toString(16);
            this.status += ''+this.fullDumpCurAt.toString(16)+"...";
            console.log("downloadFullDumpFragment is requesting " + fullAdr + "!");
            let url = window.device+'/api/flash/'+fullAdr;

            const retryWithDelay = () => {
                setTimeout(() => {
                this.downloadFullDumpFragment();
                }, 1000);
            };
            const nextWithDelay = () => {
                setTimeout(() => {
                this.downloadFullDumpFragment();
                }, 250);
            };

            fetch(url)
                .then(response => response.arrayBuffer())
                .then(buffer => {
                    if(buffer.byteLength != this.fullDumpChunkSize) {
                        this.status += "Unexpected response length (" + buffer.byteLength + "). Retrying...";
                        retryWithDelay();
                        return;
                    }
                    this.fullDumpErrors = 0;
                    this.fullDumpData = this.appendArrayBuffers(this.fullDumpData, buffer);
                    this.fullDumpCurAt += this.fullDumpChunkSize;
                    if(this.fullDumpCurAt >= (this.fullDumpFlashStart+this.fullDumpFlashSize)){
                        this.onFullDumpReadyForDownload();
                        return;
                    }
                    nextWithDelay();
                })
                .catch(err => {
                        this.fullDumpErrors ++;
                        if(this.fullDumpErrors > 100) {
                            console.error("Fragment error, too many failed attempts, stopping!");
                            console.error(err);
                            this.status += "Error: too many failed attempts.<br/>";
                            this.fullDumpRunning = 0;
                        } else {
                            console.warn("Fragment error, will retry!");
                            console.warn(err);
                            this.status += "Error; retrying...";
                            retryWithDelay();
                        }
                    }); // Never forget the final catch!
        },
	 readCustom(cb) {
	    let offset = prompt("Enter the offset (in hex):", "0x0");
	    let length = prompt("Enter the length (in bytes):", "256");
	    offset = parseInt(offset, 16);
	    length = parseInt(length);

	    if (isNaN(offset) || isNaN(length)) {
	        this.status += '<br/>Invalid offset or length entered.';
	        return;
	    }

	    this.status += `<br/>Reading custom data from offset 0x${offset.toString(16)} with length ${length}...`;
	    let url = window.device + `/api/flash/${offset.toString(16)}-${length.toString(16)}`;
	    console.log('Will use URL ' + url);

	    fetch(url)
	        .then(response => response.arrayBuffer())
	        .then(buffer => {
	            this.configdata = buffer;
	            console.log('received ' + buffer.byteLength);
	            this.status += '..received custom data...';
	            this.dump(buffer);
	            if (cb) cb();
	        })
	        .catch(err => console.error(err)); // Never forget the final catch!
	},

	downloadCustom(cb) {
	    let offset = prompt("Enter the offset (in hex):", "0x0");
	    let length = prompt("Enter the length (in bytes):", "256");
	    offset = parseInt(offset, 16);
	    length = parseInt(length);

	    if (isNaN(offset) || isNaN(length)) {
	        this.status += '<br/>Invalid offset or length entered.';
	        return;
	    }

	    this.status += `<br/>Downloading custom data from offset 0x${offset.toString(16)} with length ${length}...`;
	    let url = window.device + `/api/flash/${offset.toString(16)}-${length.toString(16)}`;
	    console.log('Will use URL ' + url);

	    fetch(url)
	        .then(response => response.arrayBuffer())
	        .then(buffer => {
	            this.downloadArrayBuffer(buffer, `custom_${offset.toString(16)}-${length}.bin`);
	            this.status += `<br/>Custom data is ready.`;
	            if (cb) cb();
	        })
	        .catch(err => console.error(err));
	},
        downloadFullDump() {
            if(0){
                alert("Not available yet.");
                return;
            }
			let rep = prompt("Are you certain? This option is slow and may crash OpenBeken on older builds, requiring a manual power-cycle. It may also reboot several times on newer builds. If you only need settings, prefer downloading the configuration partition and a LittleFS backup (.tar). Do not flash a full 2MB dump from one device to another; RF/WiFi calibration is device-specific. Type yes to continue.", "no");
			if (rep != null) {
				  if(rep[0] == "y") {
				  } else {
                    return;
                  }
			} else {
                return;
            }
            this.startDumpJob("full 2MB dump", "FullDump", 0, 2097152, "QIO");
        },
        doFlashDumpInternal() {
            if(this.fullDumpRunning!=0){
                alert("A flash dump is already in progress. Please wait for it to finish.");
                return;
            }
            // start with empty data
            this.fullDumpRunning = 1;
            console.log("doFlashDumpInternal started!");
            this.status += `<br/>Downloading ${this.getDumpJobLabel()}...`;
            this.fullDumpData = new ArrayBuffer();
            this.fullDumpCurAt = this.fullDumpFlashStart;
            this.fullDumpChunkSize = 4096;
            this.fullDumpErrors = 0;
            this.downloadFullDumpFragment();
        },
        rf(cb){
            this.status += '<br/>Reading RF configuration...';
            let url = window.device+'/api/flash/'+this.getRFAddress();
            console.log('Will use URL '+url);
            fetch(url)
                .then(response => response.arrayBuffer())
                .then(buffer => {
                    this.rfdata = buffer;
                    console.log('received '+buffer.byteLength);
                    this.status += '..received RF configuration...';
                    this.dump(buffer);
                    if(cb) cb();
                })
                .catch(err => console.error(err)); // Never forget the final catch!
        },
        getTuyaConfigAddress(){
            if(this.chipset === "BK7231T") {
				return 0x1EE000;
		}
            if(this.chipset === "BK7231N") {
				return 0x1EE000;
		}
		return 0x1D8000;
        },
        readTuyaConfig(cb){
            this.status += '<br/>Reading Tuya GPIO configuration...';
            let url = window.device+'/api/flash/1EE000-1000';
            console.log('Will use URL '+url);
            fetch(url)
                .then(response => response.arrayBuffer())
                .then(buffer => {
                    this.configdata = buffer;
                    console.log('received '+buffer.byteLength);
                    this.status += '..received Tuya GPIO configuration...';
                    this.dump(buffer);
                    if(cb) cb();
                })
                .catch(err => console.error(err)); // Never forget the final catch!
        },
        downloadTuyaConfig() {
            // it ends at 2097152 - at 2MB
            this.startDumpJob("Tuya GPIO config", "TuyaConfig", 0x1EE000, 73728, "TuyaConfig");
        },
        config(cb){
            this.status += '<br/>Reading OBK configuration...';
            let url = window.device+'/api/flash/'+this.getConfigAddress();
            console.log('Will use URL '+url);
            fetch(url)
                .then(response => response.arrayBuffer())
                .then(buffer => {
                    this.configdata = buffer;
                    console.log('received '+buffer.byteLength);
                    this.status += '..received OBK configuration...';
                    this.dump(buffer);
                    if(cb) cb();
                })
                .catch(err => console.error(err)); // Never forget the final catch!
        },
        restore_rf(cb){
			  let rep = prompt("Use this option only if your RF partition is already corrupted (for example, your MAC ends with 00 00). Otherwise, WiFi performance may decrease because the recreated RF partition uses generic calibration data. Type yes to continue.", "no");
			  if (rep != null) {
				  if(rep == "yes") {
					this.restore_rf_internal(cb);
				  }
			  }

		},
		/**
		 * Returns a random integer between min (inclusive) and max (inclusive).
		 * The value is no lower than min (or the next integer greater than min
		 * if min isn't an integer) and no greater than max (or the next integer
		 * lower than max if max isn't an integer).
		 * Using Math.round() will give you a non-uniform distribution!
		 */
		getRandomInt(min, max) {
			min = Math.ceil(min);
			max = Math.floor(max);
			return Math.floor(Math.random() * (max - min + 1)) + min;
		},

		base64ToUint8Array(b64) {
			// atob() returns a "binary string" (each char code is a byte 0..255).
			const bin = atob(b64);
			const out = new Uint8Array(bin.length);
			for (let i = 0; i < bin.length; i++) {
				out[i] = bin.charCodeAt(i) & 0xFF;
			}
			return out;
		},
		getRestoreRFTemplateBase64() {
			// Keep this list in sync with isRestoreRFSupported.
			if (this.chipset === "BK7231T" || this.chipset === "BK7231U" || this.chipset === "BK7252") {
				return "VExWAOABAAAAERERWgAAAAEREREEAAAATmG8AAIREREGAAAAhONCsmG/AxEREQQAAAAdAQAABBEREQQAAACOFVMBBREREQQAAAAbAAAABhEREQQAAABVAFQDBxEREQgAAAAQAAAAEAAAAAAiIiKiAAAAASIiIgQAAAAPAAAAAiIiIg4AAAAPj4+Ojo6NjY2Mi4sKigMiIiIOAAAAEZGRkI+Pjo2NjIuKCYkEIiIiDgAAAA6Ojo2MjIuKiomIhwaGBSIiIgQAAAACAAAABiIiIgQAAAADAAAAByIiIigAAACQkJCQkI+Pj4+Pjo6Ojo6NjY2NDIyMjIyMjIyMjIyMjIyMjIyMjIyMCCIiIgQAAAABAAAAADMzM4QAAAABMzMzBAAAAAgAAAACMzMzBAAAAAoAAAADMzMzBAAAAAkAAAAEMzMzBAAAAAMCAAAFMzMzBAAAAPgBAAAGMzMzBAAAAOgDAAAHMzMzBAAAAP8DAAAIMzMzBAAAABAAAAAJMzMzBAAAABAAAAAKMzMzBAAAAAACAAALMzMzBAAAAAACAAAAREREQAAAAAFEREQgAAAAgICAgHiAeIBwhHCEYIxYlFaUVpRUlFSUVZVUlVWVVJUCREREBAAAAO4BAAADREREBAAAAAcAAAA=";
			}
			if (this.chipset === "BK7231N" || this.chipset === "BK7231M" || this.chipset === "BK7258") {
				return "VExWAGIAAAAAERERWgAAAAEREREEAAAATmG8AAIREREGAAAAOB+NOFcOAxEREQQAAABeAQAABBEREQQAAACOFVMBBREREQQAAAAMAAAABhEREQQAAABVAFQDBxEREQgAAACAAAAAgAAAAP////////8=";
			}
			if (this.chipset === "BK7238" || this.chipset === "BK7252N") {
				return "VExWAOABAAAAERERWgAAAAEREREEAAAATmG8AAIREREGAAAAPAtZAAAAAxEREQQAAAACAQAABBEREQQAAACOFVMBBREREQQAAAA0AAAABhEREQQAAAByBOkIBxEREQgAAACAAAAAgAAAAAAiIiKiAAAAASIiIgQAAAAPAAAAAiIiIg4AAABIycnJyspKysrJyclIyAMiIiIOAAAAU9TU1NXVVdXV1NTUU9MFIiIiBAAAAAQAAAAHIiIiKAAAAJ+fn5+foKCgoKCgoKCgoKGhoaEhoaGhoaGhoaGhoaCgoKCgoKCgoJ8IIiIiBAAAAAEAAABaWlpaWlpaWlpaWlpaWlpaWlpaWlpaWlpaWlpaWlpaWlpaADMzM4QAAAABMzMzBAAAAAAAAAACMzMzBAAAAAAAAAADMzMzBAAAAAAAAAAEMzMzBAAAALwHAAAFMzMzBAAAANoHAAAGMzMzBAAAAP8PAAAHMzMzBAAAAI4PAAAIMzMzBAAAAFoAAAAJMzMzBAAAAFgAAAAKMzMzBAAAALsHAAALMzMzBAAAAAEIAAAAREREQAAAAAFEREQgAAAAeIB4gHCIdIRwiGyIWJBQmFCYTJhMmEyYTJhMmEyYTZkCREREBAAAAFcDAAADREREBAAAAAgBAAA=";
			}
			if (this.chipset === "BK7236") {
				return "VExWAPABAAAAERERWgAAAAEREREEAAAATmG8AAIREREGAAAATKkZAAAAAxEREQQAAAAkAgAABBEREQQAAACOFVMBBREREQQAAABHAAAABhEREQQAAAAAAAAABxEREQgAAACUAAAAkwAAAAAiIiKyAAAAASIiIgQAAAAPAAAAAiIiIhIAAAAAAIhBC4uLi4uLCoqKioqKCYkDIiIiEgAAAAAAcEEgoKCfn58enp6dnZ0cnAQiIiISAAAAAABgQZmZmZqbm5uampmXl5eXBSIiIgQAAAAEAAAABiIiIgQAAAAEAAAAByIiIiwAAAAAAMBAra2tra2tra2traysrKysq6urqyqqqqqqqqqqqqqqqampqamoqKiopwgiIiIEAAAAAQAAAAAzMzOEAAAAATMzMwQAAAAEAAAAAjMzMwQAAAAPAAAAAzMzMwQAAAAQAAAABDMzMwQAAAAvCAAABTMzMwQAAAAdCAAABjMzMwQAAADfDwAABzMzMwQAAAD/DwAACDMzMwQAAACSAAAACTMzMwQAAACSAAAACjMzMwQAAAAaCAAACzMzMwQAAAAACAAAAEREREAAAAABREREIAAAAIF8gXyBfIF8hXiFeIdwiHSQbJBsjG2Obo5ujm6Obo5uAkRERAQAAAAAAgAAA0RERAQAAAAAAAAA";
			}
			return null;
		},

        restore_rf_internal(cb){
            console.log('restore rf ');
            let rfRange = this.getRFAddress();
            let url = window.device+'/api/flash/'+rfRange;
            console.log('Will use URL '+url);

			// RF templates ported from BK7231Flasher/RFPartitionUtil.cs (generic TLV RF blobs).
			// We keep the first 3 bytes (OUI) fixed and randomize only the last 3 bytes of the MAC.
			const tmplB64 = this.getRestoreRFTemplateBase64();
			const correct_rf_config = tmplB64 ? this.base64ToUint8Array(tmplB64) : null;


            if(!correct_rf_config || correct_rf_config.length < 42) {
                this.status += '<br/>Restore RF is not supported on ' + this.chipset + '.';
                return;
            }

            // Status: show matched chipset + RF flash range being written.
            let rfParts = (rfRange || '').split('-');
            let rfStart = (rfParts.length > 0) ? rfParts[0] : '';
            let rfLen = (rfParts.length > 1) ? rfParts[1] : '';
            let rfStartFmt = rfStart ? ((rfStart.indexOf('0x') === 0) ? rfStart : ('0x'+rfStart)) : '';
            let rfLenFmt = rfLen ? ((rfLen.indexOf('0x') === 0) ? rfLen : ('0x'+rfLen)) : '';
            this.status += '<br/>Restoring RF configuration for ' + this.chipset + ' to ' + rfStartFmt + ' (len ' + rfLenFmt + ')...';

            // Pad to full sector so we write deterministic content (device expects 0x1000 RF partition).
            let streamData = new Uint8Array(0x1000);
            streamData.fill(0xFF);
            streamData.set(correct_rf_config, 0);

            // NOTE: MAC starts at offset 36, so last 3 bytes are 39..41
            streamData[39] = this.getRandomInt(0,255);
            streamData[40] = this.getRandomInt(0,255);
            streamData[41] = this.getRandomInt(0,255);

            if (streamData){
                fetch(url, {
                        method: 'POST',
                        body: streamData
                    })
                    .then(response => response.text())
                    .then(text => {
                        console.log('received '+text);
                        this.status += '<br/>RF configuration restored for ' + this.chipset + ' at ' + rfStartFmt + ' (len ' + rfLenFmt + '). Reboot the device.';
                    })
                    .catch(err => console.error(err)); // Never forget the final catch!
            }
        },
        dump(buffer){
            this.display = '<pre>';
            const view = new Uint8Array(buffer);

            let linestart = '0000: ';
            let lineend = '';

            for (let i = 0; i < buffer.byteLength; i++){
                if (i && !(i % 32)){
                    this.display += (linestart + ' : '+lineend + '\n');
                    linestart = ('000'+i.toString(16)).slice(-4) +': ';
                    lineend = '';
                    this.display += '\n';
                }
                linestart += ('0'+view[i].toString(16)).slice(-2) + ' ';
                lineend += (view[i] < 0x20)? '.':String.fromCharCode(view[i]);
            }
            this.display += linestart + ' : '+lineend + '\n';
            this.display += '</pre>';
        }

    },
    mounted (){

        this.rfurl = window.device+'/api/flash/'+this.getRFAddress();
        this.configurl = window.device+'/api/flash/'+this.getConfigAddress();

        console.log('mounted ota');
        this.getinfo();
    }
  }
//@ sourceURL=/vue/flash.vue
</script>

<style scoped>
.display pre{
    font-family: 'Courier New', Courier, monospace;
    font-size: 14px;
}
.flashIntro {
    margin: 0 0 12px 0;
}
.flashIntro p {
    margin: 0 0 6px 0;
}
.flashIntro code,
.jobStatusLog code {
    font-family: 'Courier New', Courier, monospace;
    padding: 0 4px;
    border-radius: 3px;
    background: #f2f2f2;
    border: 1px solid #ddd;
}

.jobStatus {
    margin: 10px 0 0 0;
}
.jobStatus h4 {
    margin: 8px 0 6px 0;
}
.jobStatusLog {
    font-size: 13px;
    line-height: 1.3;
}

.rfWriteBlock,
.cfgWriteBlock {
    display: block;
}
.rfWriteBlock .rfWriteLabel label,
.cfgWriteBlock .cfgWriteLabel label {
    display: block;
    margin: 0 0 4px 0;
}
.rfWriteBlock .rfWriteFile input[type="file"],
.cfgWriteBlock .cfgWriteFile input[type="file"] {
    display: block;
    margin: 0 0 6px 0;
    max-width: 100%;
}
.rfWriteBlock .rfWriteButton button,
.cfgWriteBlock .cfgWriteButton button {
    display: block;
    margin: 0;
    max-width: 100%;
}

.my-table,
.my-table th,
.my-table td {
    border: 1px solid black;
    border-collapse: collapse;
}
</style>
