<template>
    <div class="vue-webpack-basic p-3">
        <button v-if="recording === false" class="btn btn-primary" @click="startRecording">
            Record
        </button>
        <button v-if="recording === true && !!recorder" class="btn btn-primary" @click="stopRecording">
            Stop ({{ recordTimeSec }}s)
        </button>
        <div class="log" v-html="logs.join('<br>')">
        </div>

        <div ref="recordingsList"></div>
    </div>
</template>
<script>
import { mapActions, mapMutations, mapGetters } from 'vuex';

import 'vendor/WebAudioRecorder/WebAudioRecorder';
// import fs from 'fs';
// import 'vendor/WebAudioRecorder/WebAudioRecorder';
// import 'vendor/WebAudioRecorder/WebAudioRecorderMp3';

// webkitURL is deprecated but nevertheless
const URL = window.URL || window.webkitURL;
// shim for AudioContext when it's not avb.
const AudioContext = window.AudioContext || window.webkitAudioContext;
let audioContext; // new audio context to help us record


export default {
    components: {},
    filters: {},
    props: {},
    data(){
        return {
            formats: '',
            recording: false,
            logs: [],
            recordTimeSec: 0,
        };
    },
    computed: {
        ...mapGetters([
        ]),
        route(){
            return this.$route;
        },
    },
    watch: {
    },
    created(){},
    mounted(){
    },
    updated(){},
    destroyed(){},
    methods: {
        ...mapActions({}),
        ...mapMutations({
            SetPageSetting: 'SetPageSetting',
            CheckAdBlock: 'CheckAdBlock',
        }),
        startRecording(){
            const that = this;
            that.recording = true;
            that.recorder = null;
            window.navigator.getUserMedia = window.navigator.getUserMedia || window.navigator.webkitGetUserMedia || window.navigator.mozGetUserMedia || window.navigator.msGetUserMedia;

            /*
                Simple constraints object, for more advanced features see
                https://addpipe.com/blog/audio-constraints-getusermedia/
            */

            const constraints = {
                audio: true,
                video: false,
            };
            const encodingType = 'mp3';


            window.navigator.mediaDevices.getUserMedia(constraints).then((stream) => {
                that.log('getUserMedia() success, stream created, initializing WebAudioRecorder...');
                /*
                    create an audio context after getUserMedia is called
                    sampleRate might change after getUserMedia is called, like it does on macOS when recording through AirPods
                    the sampleRate defaults to the one set in your OS for your playback device
                */

                that.audioContext = new AudioContext({ sampleRate: 160000 });
                that.log(`Format: 2 channel ${encodingType} @ ${that.audioContext.sampleRate / 1000}kHz`);

                // assign to gumStream for later use
                that.gumStream = stream;

                /* use the stream */
                that.input = that.audioContext.createMediaStreamSource(stream);

                that.recorder = new window.WebAudioRecorder(that.input, {
                    workerDir: 'dist/js/vendor/WebAudioRecorder/', // must end with slash
                    encoding: encodingType,
                    numChannels: 2, // 2 is the default, mp3 encoding supports only 2
                    onEncoderLoading(recorder, encoding){
                        // show "loading encoder..." display
                        that.log(`Loading ${encoding} encoder...`);
                    },
                    onEncoderLoaded(recorder, encoding){
                        // hide "loading encoder..." display
                        console.log(`${encoding} encoder loaded`);
                    },
                });

                that.recorder.onComplete = function(recorder, blob){
                    that.log('Encoding complete');
                    that.createDownloadLink(blob, recorder.encoding);
                };

                that.recorder.setOptions({
                    timeLimit: 600,
                    encodeAfterRecord: true,
                    ogg: { quality: 1 },
                    mp3: { bitRate: 320 },
                });

                that.recordTimeSec = 0;
                // start the recording process
                that.recorder.startRecording();

                that.recorderTimer = setInterval(() => {
                    that.recordTimeSec += 1;
                }, 1000);


                that.recorder.onTimeout = function(recorder){
                    console.log('onTimeout');
                };
                that.recorder.onEncodingProgress = function(recorder, progress){
                    console.log('onEncodingProgress', progress);
                };
                that.recorder.onEncodingCanceled = function(recorder){
                    console.log('onEncodingCanceled');
                };
                that.recorder.onError = function(recorder, message){
                    console.log('onError', message);
                };
            }).catch((err) => {
                // // enable the record button if getUserMedia() fails
                // recordButton.disabled = false;
                // stopButton.disabled = true;
                // pauseButton.disabled = true;
            });
        },
        stopRecording(){
            const that = this;
            that.recording = false;
            clearInterval(that.recorderTimer);
            that.log('stopRecording() called');

            // stop microphone access
            that.gumStream.getAudioTracks()[0].stop();


            // tell the recorder to finish the recording (stop recording + encode the recorded audio)
            that.recorder.finishRecording();
            that.recorder = null;
        },
        createDownloadLink(blob, encoding){
            const that = this;
            that.log('createDownloadLink', blob, encoding);
            const url = URL.createObjectURL(blob);
            const au = document.createElement('audio');
            const li = document.createElement('li');
            const link = document.createElement('a');

            // add controls to the <audio> element
            au.controls = true;
            au.src = url;

            // link the a element to the blob
            link.href = url;
            link.download = `${new Date().toISOString()}.${encoding}`;
            link.innerHTML = link.download;

            // add the new audio and a elements to the li element
            li.appendChild(au);
            li.appendChild(link);

            // add the li element to the ordered list
            that.$refs.recordingsList.appendChild(li);
        },
        log(text){
            const that = this;
            const logs = JSON.parse(JSON.stringify(that.logs));
            logs.push(text);
            that.logs = logs;
        },
    },
};
</script>
<style lang="scss" scoped>
</style>