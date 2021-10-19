<template>
  <v-app>
	<v-app-bar
      app
      color="primary"
      dark
    >
      <v-toolbar-title>{{ $route.name }}</v-toolbar-title>
    </v-app-bar>   
    <v-main>  	
		<v-container>
			<v-row>
				<v-col cols="12" align="center">
					<div id="webcam-container"></div>
				</v-col>
			</v-row>
			<v-row>
				<v-col cols="12">
					<v-card v-if="resultItem" >
						<v-card-text>
							<div class="text-h4 black--text font-weight-bold">{{ resultItem.name }}</div>
							<v-row title>
								<v-spacer></v-spacer>
								<v-col shrink></v-col>
								<v-col shrink primary--text pl-3>{{ resultItem.message }}</v-col>
							</v-row>
							<v-divider class="my-6"></v-divider>
							<v-row title>
								<v-col xs4 grey--text></v-col>
								<v-col>{{ resultItem.m1 }}</v-col>
							</v-row>
							<v-row title mt-2>
								<v-col xs4 grey--text></v-col>
								<v-col>{{ resultItem.m2 }}</v-col>
							</v-row>
							<v-row title mt-2>
								<v-col xs4 grey--text></v-col>
								<v-col>{{ resultItem.m3 }}</v-col>
							</v-row>
							<v-row title mt-2>
								<v-col xs4 grey--text></v-col>
								<v-col>{{ resultItem.m4 }}</v-col>
							</v-row>
						</v-card-text>
					</v-card>
					<v-card v-show="resultItem === null" class="fill-height">
						<v-row fill-height justify-center align-center row wrap>
							<v-col shrink xs12 text-center>
								<v-img :style="{ margin: '0 auto' }" :width="100" :src="require('@/assets/smail.png')"></v-img>								
							</v-col>
							<v-col shrink xs12 text-center>								
								<div class="grey--text text--darken-1 mt-2">자신의 모습을 카메라에 보여주세요~!</div>
							</v-col>
						</v-row>
					</v-card>
				</v-col>
			</v-row>
		</v-container>
	</v-main>	
  </v-app>
</template>
<script>
// eslint-disable-next-line no-unused-vars
import * as tf from '@tensorflow/tfjs';
import * as tmImage from '@teachablemachine/image';
import data from '../assets/data.json'
export default {
	name: 'rlas',
	data: () => ({
		URL: 'https://teachablemachine.withgoogle.com/models/fM-iX7PQt/', //				
		webcam: null,
		labelContainer: null,
		maxPredictions: null,
		resultItem: null,
		msgURL: 'https://slack.com/api/chat.postMessage',		
		msgToken1 : '2541818091588',
		msgToken2 : '2541834361300',
		msgToken3 : 'bWMPkMTpj9uVnSukq9NBcFyt',
		msgChannel : 'class',
		msgUser: null,
		msgText: null,
		state_count: -1,
	}),
	created () {
		this.init()
	},
	methods: {
		async init () {
			const modelURL = this.URL + "model.json";
			const metadataURL = this.URL + "metadata.json";
			// load the model and metadata
			// Refer to tmImage.loadFromFiles() in the API to support files from a file picker
			// or files from your local hard drive
			// Note: the pose library adds "tmImage" object to your window (window.tmImage)
			this.model = await tmImage.load(modelURL, metadataURL);
			this.maxPredictions = this.model.getTotalClasses();
			// Convenience function to setup a webcam
			const flip = true; // whether to flip the webcam
			const width = 376
			this.webcam = new tmImage.Webcam(width, width, flip); // width, height, flip
			await this.webcam.setup(); // request access to the webcam
			await this.webcam.play();
			window.requestAnimationFrame(this.loop);
			// append elements to the DOM
			document.getElementById("webcam-container").appendChild(this.webcam.canvas);
			let cookie = this.$cookies.get('name');      
			this.msgUser = cookie			
			await this.post_message(this.msgUser+'님이 입장하였습니다.');
		},
		async loop () {
			this.webcam.update(); // update the webcam frame
			await this.predict();
			window.requestAnimationFrame(this.loop);
		},
		async predict () {
			// predict can take in an image, video or canvas html element
			const prediction = await this.model.predict(this.webcam.canvas);
			let isPrediction = false;						
			let max_prob = prediction[0].probability.toFixed(2);
			let max_index = 0;
			// 4개의 클래스 중 최대 확률인 클래스 찾기
			for (let i = 1; i < this.maxPredictions; i++) {   
				if (max_prob < prediction[i].probability.toFixed(2)) {
					max_prob = prediction[i].probability.toFixed(2);
					max_index = i;
				}
			}
			// 최대 확률의 클래스가 0.5 이상이면 찾은 것으로 함
			if( max_prob > 0.5) {
				if(max_index === 0) {
					this.state_count = 0;
				} else {
					this.state_count++;	 // 바른 자세가 아닌 경우 count함				
				}				
				console.log(this.state_count);
				const item = data[prediction[max_index].className];				
				this.resultItem = item
				this.resultItem.m1 = this.state_count;  // for testing
				isPrediction = true
			}
			if (this.state_count === -1 && isPrediction === false) {  // 기존은 예측을 못할 경우 아이콘이 보였으나, 처음에만 나오도록 설정
				this.resultItem = null
			}			
			if(this.state_count > 1000) {    // 바른 자세가 연속적으로 1000번 이상 아닌 경우 그 때의 상태를 메시지를 보냄
				let text = this.msgUser + "님, " + this.resultItem.message;
				await this.post_message(text);
				this.state_count = 0;
				this.resultItem.m1 = this.state_count;   // for testing
			}
		},
		// slack에 메시지를 보내는 함수
		async post_message(text) {
			let date = new Date();
			let parmas = new URLSearchParams();			
			this.msgText = "["+date.getHours()+":"+date.getMinutes()+":"+date.getSeconds()+"/"+this.msgUser+"]"+text;
			parmas.append("token", 'x'+'o'+'x'+'b-' + this.msgToken1+'-'+ this.msgToken2+'-'+ this.msgToken3);
			parmas.append("channel", this.msgChannel);
			parmas.append("text", this.msgText);			
			this.$http.post(this.msgURL, parmas)
				.then(function(response)  {
					console.log(response);
				});
		}
   }
}
</script>
<style>
</style>
