<template>
    <div>
        <base-header type="gradient-success" class="pb-6 pb-8 pt-5 pt-md-8">
            <!-- Card stats -->
            <div class="row">
                
            </div>
        </base-header>

        <div class="container-fluid mt--7">
            <div class="row">
                <div class="col-xl-6 mb-5 mb-xl-0">
                    <card header-classes="bg-transparent">
                        <div slot="header" class="row align-items-center">
                            <div class="col">
                                <h6 class="text-light text-uppercase ls-1 mb-1">View</h6>
                                <h5 class="h3 mb-0">Image View</h5>
                            </div>
                            <div class="col">
                                <ul class="nav nav-pills justify-content-end">
                                    <li class="nav-item mr-2 mr-md-0">
                                          <a class="nav-link py-2 px-3"
                                           href="#">
                                            <input type="file" @change="onFileChange(item, $event)">
                                        </a>
                                    </li>
                                </ul>
                            </div>
                        </div>
                        <div v-if="imageShow == null">
                            <img src="/svgs/not_connected.svg" class="img-fluid">
                            <h5 class="h3 mt-3 mb-0 text-center">Please upload image</h5>
                        </div>
                        <div v-else>
                            <img class="img-fluid" :src="imageShow" alt="" >
                        </div>
                        <div class="col mt-2" v-if="imageShow!=null">
                            <ul class="nav nav-pills justify-content-end">
                                <li class="nav-item mr-2 mr-md-0">
                                    <a class="nav-link py-2 px-3"
                                        href="#"
                                        @click.prevent="predictData">
                                        <span class="d-none d-md-block">Predict</span>
                                    </a>
                                </li>
                            </ul>
                        </div>
                    </card>
                </div>
                
                <div class="col-xl-6">
                    <div class="row">
                        <div class="col-xl-12">
                            <card header-classes="bg-transparent">
                                <div slot="header" class="row align-items-center">
                                    <div class="col">
                                        <h6 class="text-uppercase text-muted ls-1 mb-1">Predict</h6>
                                        <h5 class="h3 mb-0">Label Predicted</h5>
                                       
                                    </div>
                                </div>
                                
                                <div>
                                    <div class="row">
                                        <div class="col-12">
                                            <h2>Prediction</h2>
                                        </div>
                                        <div class="col-12">
                                            Normal : {{this.prediction[0]}}
                                        </div>
                                        <div class="col-12">
                                            Pneumonia : {{this.prediction[1]}}
                                        </div>
                                    </div>
                                </div>
                            </card>
                        </div>
                    </div>
                </div>
            </div>
            
        </div>

    </div>
</template>
<script>
  // Charts
    import * as chartConfigs from '@/components/Charts/config';
    import LineChart from '@/components/Charts/LineChart';
    import BarChart from '@/components/Charts/BarChart';
    import { format } from 'date-fns'
    // Tables
    import SocialTrafficTable from './Dashboard/SocialTrafficTable';
    import PageVisitsTable from './Dashboard/PageVisitsTable';
    import axios from 'axios';
    import * as tf from '@tensorflow/tfjs';

    export default {
        components: {
        LineChart,
        BarChart,
        PageVisitsTable,
        SocialTrafficTable,
        },
        sockets: {  
        connect: function () {
            this.cctvState = true;
        },
        image: function (data) {
            this.cctvState = true;
            this.image = data;
        },
        person:function(data){
            console.log(data);
            this.captureData();
        },
        connect_error:function(){
            this.cctvState = false;
            this.image = '';
        },
        disconnect:function(){
            this.cctvState = false;
            this.image = '';
        }
        },
    
        data() {
        return {
            image:'',
            license:'',
            capturedImage:'',
            capturedLicense:'',
            cctvState:false,
            meta:null,
            licenseProfile:{
                meta:null,
                province:null,
                taxYear:{
                    month:null,
                    year:null
                }
            },
            imageShow:null,
            taxStatus:null,
            owner:null,
            model:null,
            prediction:[]
        };
        },
        methods: {
            captureData(){
                this.capturedImage = this.image;
                let imageData = `data:image/jpeg;base64,${this.capturedImage}`;
                axios.post('http://localhost:8000/api/postImage', {
                    imageData:this.capturedImage
                }).then(res => {})
            },
            getFormattedDate(date){
                var dateSplitted = date.split('-')
                return format(new Date(dateSplitted[0],dateSplitted[1],dateSplitted[2]), 'dd MMMM yyyy')
            },
            onFileChange(item, e) {
                var files = e.target.files || e.dataTransfer.files;
                console.log(files)
                if (!files.length)
                    return;
                this.createImage(item, files[0]);
            },
            createImage(item, file) {
                var image = new Image();
                var reader = new FileReader();
                reader.onload = (e) => {
                    this.imageShow = e.target.result;
                };
                reader.readAsDataURL(file);
            },
            predictData(){
                this.prediction.length = 0
                var img = new Image();
                img.src = this.imageShow
                const xs = tf.browser.fromPixels(img);
                img = tf.image.resizeBilinear(xs,[225,225]);
                const offset = tf.scalar(255);
                const normalized = img.div(offset);
                var data = normalized.expandDims(0)
                const prediction = this.model.predict(data).data().then(prediction=>{
                    this.prediction.push(prediction[0])
                    this.prediction.push(prediction[1])
                })
            }
        },
        async mounted() {
          this.model = await tf.loadLayersModel('/model.json');
        },
        watch: {
        'license': function(val, oldVal){
            if (val != '') {
                this.captureData()
            }
        }
        }
    };
</script>
<style></style>
