<template>
	<div class="container">
		<h1 class="mb-5">{{ searchData.flag }}&nbsp;</h1>
		<div class="row justify-content-center mb-5">
			<div class="col-3">
				<ul class="list-group">
					<li
						v-for="(item, idx) in topTenObj.topTenList"
						:key="item.searchKeyword"
						class="list-group-item d-flex justify-content-between align-items-center"
					>
						<span
							class="text-truncate"
							@click="clickToptenKeyword(item.searchKeyword)"
							>{{ idx + 1 }}. {{ item.searchKeyword }}</span
						>
						<span class="badge bg-primary rounded-pill"
							>조회수 {{ item.viewCount }}</span
						>
					</li>
				</ul>
			</div>
		</div>
		<div class="row justify-content-center mb-5">
			<div class="input-group w-50 h-100">
				<input
					class="form-control"
					v-model="searchParam.query"
					placeholder="검색어를 입력해 주세요"
					@keyup.enter="search"
				/>
				<button class="btn btn-outline-secondary" type="button" @click="search">
					검색
				</button>
			</div>
		</div>
		<div class="float-end">
			<select
				class="form-select"
				aria-label="Default select example"
				v-model="searchParam.sort"
			>
				<option value="accuracy">최신순</option>
				<option value="recency">정확도순</option>
			</select>
		</div>
		<br />
		<br />
		<div v-if="searchData.flag == 'kakaoApi'" id="card_list">
			<div class="row" v-for="(row, index) in rows" :key="index">
				<div class="col-3 mb-5" v-for="item in row" :key="item.blogname">
					<div class="card h-100">
						<img class="card-img-top" :src="item.thumbnail" alt="" />
						<div
							class="card-body text-start pt-4 d-flex flex-column justify-content-between"
						>
							<h3 class="card-title mb-3 fw-bold text-truncate">
								{{ item.blogname }}
							</h3>
							<p class="card-text" v-html="item.contents"></p>
							<p>{{ dateParsing(item.datetime) }}</p>
							<button
								type="button"
								class="btn btn-warning"
								@click="openUrl(item.url)"
							>
								방문하기
							</button>
						</div>
					</div>
				</div>
			</div>
		</div>
		<div v-else>
			<div class="row" v-for="(row, index) in rows" :key="index">
				<div class="col-3 mb-5" v-for="item in row" :key="item.blogname">
					<div class="card h-100">
						<img class="card-img-top" :src="item.thumbnail" alt="" />
						<div
							class="card-body text-start pt-4 d-flex flex-column justify-content-between"
						>
							<h3
								class="card-title mb-3 fw-bold text-truncate"
								v-html="item.title"
							></h3>
							<h5>{{ item.bloggername }}</h5>
							<p class="card-text" v-html="item.description"></p>
							<p>{{ dateParsing(item.postdate) }}</p>
							<button
								type="button"
								class="btn btn-warning"
								@click="openUrl(item.link)"
							>
								방문하기
							</button>
						</div>
					</div>
				</div>
			</div>
		</div>
		<div id="no-content" v-if="!pageCount">검색 결과가 없습니다.</div>
		<nav aria-label="..." v-if="pageCount">
			<ul
				class="pagination d-flex h-100 justify-content-center align-items-center mb-5"
			>
				<li
					class="page-item"
					:class="{ disabled: pageNation.currentPage == 1 }"
				>
					<a class="page-link" @click="pagingUtil.prePage">Previous</a>
				</li>
				<li class="page-item active">
					<a class="page-link">{{ pageNation.currentPage }}</a>
				</li>
				<li class="page-item">
					<a
						class="page-link"
						@click="pagingUtil.nextPage"
						v-if="pageNation.nextPage"
						>{{ pageNation.nextPage }}
					</a>
				</li>
				<li class="page-item">
					<a
						class="page-link"
						@click="pagingUtil.nextPage"
						v-if="pageNation.nextPage"
						>Next</a
					>
				</li>
			</ul>
		</nav>
	</div>
</template>

<script>
import { computed, defineComponent, reactive } from "vue";
import axios from "axios";

export default defineComponent({
	name: "BlogSearch",
	setup() {
		const searchParam = reactive({
			query: "",
			sort: "accuracy",
			page: 1,
			size: 10,
		});

		const searchData = reactive({
			content: [],
			flag: "",
		});

		const topTenObj = reactive({
			topTenList: [],
		});

		const pageNation = reactive({
			currentPage: 1,
			nextPage: 0,
		});

		const getSearch = () => {
			if (!searchParam.query.trim()) {
				return;
			}
			axios
				.get("http://localhost:8080/api/search", { params: searchParam })
				.then(res => {
					responseDataParsing(res.data);
					pagingParsing(res.data);
					getTopTenList();
				})
				.catch(err => {
					console.log("err: ", err);
				});
		};

		const getTopTenList = () => {
			axios
				.get("http://localhost:8080/api/topten")
				.then(res => {
					topTenObj.topTenList = res.data.topTenList;
				})
				.catch(err => {
					console.log("err: ", err);
				});
		};

		getTopTenList();

		const clickToptenKeyword = data => {
			searchParam.query = data;
		};

		const search = () => {
			resetPage();
			getSearch();
		};

		const resetPage = () => {
			searchParam.page = 1;
		};

		const pagingParsing = data => {
			pageNation.currentPage = data.page;
			if (data.is_end) {
				pageNation.nextPage = 0;
			} else {
				if (data.page == 50) {
					pageNation.nextPage = 0;
				} else {
					pageNation.nextPage = data.page + 1;
				}
			}
		};

		const pagingUtil = {
			prePage: () => {
				searchParam.page = pageNation.currentPage - 1;
				getSearch();
			},
			nextPage: () => {
				searchParam.page = pageNation.nextPage;
				getSearch();
			},
		};

		const pageCount = computed(() => {
			return searchData.content.length;
		});

		const rows = computed(() => {
			const rows = [];
			const itemsPerRow = 4;
			for (let i = 0; i < searchData.content.length; i += itemsPerRow) {
				rows.push(searchData.content.slice(i, i + itemsPerRow));
			}
			return rows;
		});

		const openUrl = url => {
			window.open(url);
		};

		const dateParsing = datetime => {
			if (searchData.flag == "kakaoApi") {
				const date = new Date(datetime);
				const formattedDate = date.toISOString().substring(0, 10);
				const formattedTime = date.toTimeString().substring(0, 8);

				return formattedDate + " " + formattedTime;
			} else {
				const date = new Date(
					`${datetime.substring(0, 4)}-${datetime.substring(
						4,
						6
					)}-${datetime.substring(6, 8)}`
				);
				const formattedDate = date.toISOString().substring(0, 10);

				return formattedDate;
			}
		};

		const responseDataParsing = res => {
			if (res.flag == "kakaoApi") {
				Object.assign(searchData, res);
			} else {
				searchData.content = res.content;
				searchData.flag = res.flag;
			}
		};

		return {
			searchParam,
			searchData,
			getSearch,
			rows,
			openUrl,
			dateParsing,
			responseDataParsing,
			search,
			pageNation,
			pagingParsing,
			pageCount,
			pagingUtil,
			resetPage,
			getTopTenList,
			topTenObj,
			clickToptenKeyword,
		};
	},
});
</script>
<style>
ul li {
	list-style-type: none;
}
.text-truncate {
	cursor: pointer;
}
</style>
