<template>
	<div class="main">
		<div class="form">
			<el-card shadow="never">
				<span class="title">Warning: Be careful! Don't share your user token</span>
				<p>This page only will use your user token to get your server list and server emoticons</p>
			</el-card>
			<div class="bar">
				<el-input class="input" v-model="token" placeholder="Enter your user token here" clearable />
				<el-button color="#5865F2" plain @click="dialogVisible = true"> what is token? </el-button>
				<el-button color="#5865F2" plain @click="submit"> submit </el-button>
				<el-checkbox class="checkbox" v-model="isSave" label="Remember your token" size="large" />
			</div>

			<el-dialog v-model="dialogVisible" title="How to get user token" align-center>
				<ol>
					<li>Open Discord desktop app or login Discord website</li>
					<li>Open Chrome developer tools with keyboard shortcut</li>
					<li>F12 or Ctrl + Shift + I</li>
					<li>Go to Network tab</li>
					<li>Click XHR button, only filter to XHR requests</li>
					<li>Do any action in Discord, like open a channel</li>
					<li>Click the science request that appears in the list</li>
					<li>Go to Headers tab</li>
					<li>Find Authorization in Request Headers and copy the token (make sure to copy the entire token, don't copy any spaces)</li>
				</ol>
				<img src="/discordTokenHint.jpg" alt="token hint" />
			</el-dialog>
		</div>

		<div class="list">
			<div class="serverList" v-if="serverLoad">
				<el-table
					ref="singleTableRef"
					:data="serverList"
					highlight-current-row
					height="400"
					style="width: auto"
					@current-change="handleCurrentChange"
				>
					<el-table-column type="index" width="50" />
					<el-table-column label="Server Icon" width="80">
						<template #default="scope">
							<img
								:src="`https://cdn.discordapp.com/icons/${scope.row.id}/${scope.row.icon}.png`"
								alt="Server Icon"
								style="width: 50px; height: 50px"
							/>
						</template>
					</el-table-column>
					<el-table-column property="id" label="Server ID" width="190" />
					<el-table-column property="name" label="Server Name" />
				</el-table>
			</div>

			<div class="emoticonList" v-if="emojiListLoad">
				<div class="bar">
					<el-button color="#5865F2" plain @click="downloadEmoji('all')">Download All Emoji</el-button>
					<el-button color="#5865F2" plain @click="downloadEmoji('selected')">Download Selected Emoji</el-button>
					<el-button color="#5865F2" plain @click="downloadSticker('all')">Download All Sticker</el-button>
					<el-button color="#5865F2" plain @click="downloadSticker('selected')">Download Selected Sticker</el-button>
				</div>
				<el-collapse v-model="activeNames">
					<el-collapse-item title="Emoji" name="1">
						<span class="emoction" v-for="item in emojiList" :key="item.id">
							<input type="checkbox" :id="item.id" v-model="selectedEmoji" :value="item" />
							<label :for="item.id">
								<img :src="getEmojiUrl(item.id, item.animated)" alt="emoji" />
								<span>{{ item.name }}</span>
							</label>
						</span>
					</el-collapse-item>
					<el-collapse-item title="Sticker" name="2">
						<span class="emoction" v-for="item in stickerList" :key="item.id">
							<input type="checkbox" :id="item.id" v-model="selectedSticker" :value="item" />
							<label :for="item.id">
								<img :src="getStickerUrl(item.id, item.format_type)" alt="emoji" />
								<span>{{ item.name }}</span>
							</label>
						</span>
					</el-collapse-item>
				</el-collapse>
			</div>
		</div>
	</div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import { ElMessageBox, ElNotification } from "element-plus";
import axios from "axios";
import { API } from "@/assets/discordApi.js";
import JSZip from "jszip";
import { saveAs } from "file-saver";

// 通知-错误
const notification = (message) => {
	ElNotification({
		title: "Error",
		message: message,
		type: "error",
	});
};

const token = ref("");
const dialogVisible = ref(false);
const isSave = ref(false);

// 提交token，判断是否为空并请求服务器列表
const serverList = ref([]);
let serverLoad = ref(false);
const submit = async () => {
	if (token.value == "") {
		return notification("token为空");
	}
	if (isSave.value) {
		localStorage.setItem("token", token.value);
	}

	try {
		const res = await API.request("GET", API.guilds, token.value);
		const info = await res.json();
		serverList.value = info.map(({ id, name, icon }) => ({ id, name, icon }));
		serverLoad.value = true;
	} catch (error) {
		notification("服务器列表获取失败: " + error);
	}
};

// 点击服务器, 获取表情列表
const emojiList = ref([]);
const stickerList = ref([]);
let emojiListLoad = ref(false);
const activeNames = ref(["1"]);
const handleCurrentChange = async (e) => {
	try {
		const [emojiRes, stickerRes] = await Promise.all([
			API.request("GET", API.emojis(e.id), token.value),
			API.request("GET", API.stickers(e.id), token.value),
		]);

		emojiList.value = await emojiRes.json();
		stickerList.value = await stickerRes.json();

		emojiListLoad.value = true;
	} catch (error) {
		notification("表情列表或贴纸列表获取失败: " + error);
	}
};

// 拼接emoji URL
const getEmojiUrl = (id, animated) => {
	return `https://cdn.discordapp.com/emojis/${id}.${animated ? "gif" : "png"}?v=1`;
};
// 拼接sticker URL
const getStickerUrl = (id, format) => {
	let suffix;
	switch (format) {
		case 1:
			suffix = "png";
			break;
		case 2:
			suffix = "png";
			break;
		case 3:
			suffix = "lottie";
			break;
		case 4:
			suffix = "gif";
			break;

		default:
			console.log("未知类型");
			break;
	}
	return `https://media.discordapp.net/stickers/${id}.${suffix}?size=1024`;
};

// 下载
const selectedEmoji = ref([]);
const selectedSticker = ref([]);
const downloadEmoji = (type) => {
	const zip = new JSZip();
	const fetchPromises = [];

	const emojiLists = type === "all" ? emojiList.value : selectedEmoji.value;

	emojiLists.forEach((item) => {
		const fileName = encodeURIComponent(item.name) + (item.animated ? ".gif" : ".png");
		const imageUrl = getEmojiUrl(item.id, item.animated);

		const fetchPromise = fetch(imageUrl)
			.then((response) => response.blob())
			.then((image) => zip.file(fileName, image))
			.catch((error) => console.error("Error fetching image:", error));

		fetchPromises.push(fetchPromise);
	});

	Promise.all(fetchPromises).then(() => {
		zip.generateAsync({ type: "blob" })
			.then((content) => saveAs(content, type === "all" ? "download-all-emoji.zip" : "download-selected-images.zip"))
			.catch((error) => console.error("Error generating zip:", error));
	});
};
const downloadSticker = (type) => {
	const zip = new JSZip();
	const fetchPromises = [];

	const stickerLists = type === "all" ? stickerList.value : selectedSticker.value;

	stickerLists.forEach((item) => {
		let fileName = encodeURIComponent(item.name);
		const imageUrl = getStickerUrl(item.id, item.format_type);
		switch (item.format_type) {
			case 1:
				fileName += ".png";
				break;
			case 2:
				fileName += ".png";
				break;
			case 3:
				fileName += ".lottie";
				break;
			case 4:
				fileName += ".gif";
				break;

			default:
				console.log("未知类型");
				break;
		}

		const fetchPromise = fetch(imageUrl)
			.then((response) => response.blob())
			.then((image) => {
				zip.file(fileName, image);
			})
			.catch((error) => {
				console.error("Error fetching image:", error);
			});

		fetchPromises.push(fetchPromise);
	});

	Promise.all(fetchPromises).then(() => {
		zip.generateAsync({ type: "blob" })
			.then((content) => {
				saveAs(content, "download-all-sticker.zip");
			})
			.catch((error) => {
				console.error("Error generating zip:", error);
			});
	});
};

onMounted(() => {
	const saveToken = localStorage.getItem("token");
	if (saveToken) {
		token.value = saveToken;
	}
});
</script>

<style lang="scss" scoped>
@import url("../assets/download.scss");
</style>
