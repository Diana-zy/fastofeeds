<template>
  <div class="page">
    <template v-if="newInfo">
      <Header :lang="newInfo.language" />
      <article class="article">
        <h1 class="article-title">{{ newInfo.name }}</h1>
        <div class="news-detail">{{ newInfo.first_paragraph }}</div>
        <div id="relatedsearches1"> </div>
        <NuxtImg
          format="auto"
          fit="cover"
          width="900"
          :src="newInfo.cover"
          :alt="newInfo.name"
          class="article-img"
          preload
        />
        <!-- eslint-disable vue/no-v-html -->
        <div class="news-detail">
          <template v-for="(item, index) in contentItems">
            <div
              v-if="item.type === 'content'"
              :key="`content-${index}`"
              v-html="item.content"
            ></div>
            <div v-else id="relatedsearches2" :key="`relatedsearch-${index}`"></div>
          </template>
        </div>
        <!--eslint-enable-->
      </article>
      <Footer :lang="newInfo.language" />
    </template>
    <div v-else class="mask-box"><loading /></div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      channelId: "",
      newInfo: null,
      splitTextCount: 400,
      isAdAdded: false
    };
  },
  computed: {
    contentItems() {
      const self = this;
      const parts = this.newInfo.content.split(/(<p[^>]*>.*?<\/p>)/gs);
      let charCount = 0;
      const items = [];

      parts.forEach((part, index) => {
        // 如果是最后一个段落，并且广告还没有添加，则插入到倒数第二段
        if (parts.length - 1 === index && !self.isAdAdded) {
          items.push({
            type: "ad"
          });
          self.isAdAdded = true;
        }

        if (!part.trim()) return; // 跳过空字符串

        // 添加内容
        items.push({
          type: "content",
          content: part
        });

        // 如果不是p标签，不计算字符数和插入广告
        if (!part.startsWith("<p")) return;

        if (self.isAdAdded) return;
        // 计算纯文本长度
        const textContent = part.replace(/<[^>]+>/g, "");
        charCount += textContent.length;

        if (charCount >= self.splitTextCount) {
          items.push({
            type: "ad"
          });
          // 是否已经push过广告
          self.isAdAdded = true;
        }
      });

      return items;
    }
  },
  mounted: function () {
    window.setCookie("mounted", 1);
    this.getDetailInfo();
  },
  methods: {
    async getDetailInfo() {
      const aid = (this.$route.params.detail3 || "").split("-").pop();
      const data = await this.$axios.$get("/api/article/detail", {
        params: {
          site_id: process.env.SITE_ID,
          article_id: aid
        }
      });
      data.content = data.content.replace(/font-family:\s*['"]?宋体['"]?;/g, "");
      data.content = data.content.replace(/<\/h4><p><br><br>|<br><br><\/p><h4>/g, (match) => {
        return match.includes("</h4><p>") ? "</h4><p>" : "</p><h4>";
      });
      this.newInfo = data;

      this.setChannelId();
      /* 确保dom更新后调用广告请求 */
      this.$nextTick(() => {
        this.newInfo.no_entry !== 1 && this.handleAdsScript();
      });
    },
    handleAdsScript() {
      const buffer = window.getCookie("pathInfo");
      if (!buffer || Number(JSON.parse(buffer)[window.location.pathname]) < 3) {
        this.addAdSenseScript();
      }
    },
    setChannelId() {
      // 获取 URL 查询参数
      const searchParams = new URLSearchParams(window.location.search);
      // AdSense 配置参数
      if (searchParams.has("channel")) {
        this.channelId = searchParams.get("channel");
      } else {
        this.channelId = this.newInfo.channel || "";
        if (this.channelId !== "") {
          searchParams.set("channel", this.channelId);
          const newUrl = `${window.location.origin}${
            window.location.pathname
          }?${searchParams.toString()}`;
          window.history.replaceState({}, "", newUrl);
        }
      }
    },
    addAdSenseScript() {
      // 获取 URL 查询参数
      const searchParams = new URLSearchParams(window.location.search);
      let terms = searchParams.has("terms") ? searchParams.get("terms") : "";
      terms = terms.replace(/[，]/g, ",");
      // 获取Url携带的headline参数
      let headline = searchParams.has("headline") ? searchParams.get("headline") : "";
      if (headline === "{title}" || headline === "{{ad_title}}") {
        headline = "";
      }

      const paramKeys = [];
      // 遍历查询参数并将其添加到 paramKeys 数组中
      for (const param of searchParams) {
        paramKeys.push(param[0]);
      }
      const ignoredPageParams = paramKeys.join(",");

      const channelId = window.getParam("channel");
      const hiSource = window.getParam("hi_source");
      const hiPc = window.getParam("hi_pc");
      const resultsPageBaseUrl = window.getResultsPageUrl({
        channel: channelId,
        from: "detail",
        hi_source: hiSource,
        hi_pc: hiPc
      });
      const adSenseConfig = {
        channel: this.channelId,
        pubId: "partner-pub-6612490456597819",
        styleId: "7767580164",
        adsafe: "low",
        ignoredPageParams,
        relatedSearchTargeting: "content",
        resultsPageBaseUrl,
        resultsPageQueryParam: "query",
        terms: terms || this.newInfo.terms,
        referrerAdCreative: headline || terms || this.newInfo.referrer_ad_creative,
        ivt: false,
        adtest: "off"
      };

      // 初始化 _googCsa 并加载相关搜索广告
      // eslint-disable-next-line no-undef
      _googCsa(
        "relatedsearch",
        adSenseConfig,
        {
          container: "relatedsearches1", // 广告容器 ID
          relatedSearches: 5, // 相关搜索广告数量
          adLoadedCallback: function (loaded, response, isExperimentVariant, callbackOptions) {
            if (response) {
              window.trackEventToPixel("D_C_AC");
              window.pushEventParamsToGtm("C_AC");
              window.setCookie("query_ad", 1);
              try {
                let numberOfKeys = 0;
                let concatenatedKeys = "miss";
                if (callbackOptions.termPositions) {
                  const keys = Object.keys(callbackOptions.termPositions);
                  numberOfKeys = keys.length;
                  concatenatedKeys = keys.join(",");
                }
                const element = document.getElementById("master-1");
                const height = parseFloat(element.style.height);
                const result = Math.round(height / 105);

                // eslint-disable-next-line no-undef
                dataLayer.push({
                  event: "C_AC_IN",
                  num: result,
                  queryNum: 5,
                  key1: numberOfKeys,
                  key2: concatenatedKeys
                }); // 事件推送到 dataLayer
              } catch (e) {
                console.log(e);
              }
            }
          }
        },
        {
          container: "relatedsearches2", // 广告容器 ID
          relatedSearches: 5, // 相关搜索广告数量
          adLoadedCallback: function (loaded, response, isExperimentVariant, callbackOptions) {
            if (response) {
              // eslint-disable-next-line no-undef
              window.pushEventParamsToGtm("C_AC_SECOND"); // 事件推送到 dataLayer
              try {
                let numberOfKeys = 0;
                let concatenatedKeys = "miss";
                if (callbackOptions.termPositions) {
                  const keys = Object.keys(callbackOptions.termPositions);
                  numberOfKeys = keys.length;
                  concatenatedKeys = keys.join(",");
                }
                const element = document.getElementById("relatedsearches2");
                const height = parseFloat(element.clientHeight);
                const result = Math.round(height / 105);

                // eslint-disable-next-line no-undef
                dataLayer.push({
                  event: "C_AC_IN_SECOND",
                  queryNum: 5,
                  num: result,
                  key1: numberOfKeys,
                  key2: concatenatedKeys
                }); // 事件推送到 dataLayer
              } catch (e) {
                console.log(e);
              }
            }
          }
        }
      );
    }
  }
};
</script>

<style lang="scss" scoped>
.article-img {
  width: 100%;
  margin-bottom: 1em;
}
.article {
  padding-bottom: 32px;
  border-bottom: 1px solid #ececee;
  min-height: calc(100vh - 72px - 56px - 64px);
}
.article-title {
  font-size: 26px;
  font-family: "rssb";
  font-weight: bold;
  line-height: 30px;
  margin-bottom: 24px;
}
.read-more {
  line-height: 4;
}
.hide {
  display: none;
  &.show {
    display: block;
  }
}
.news-box-2 {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 24px;
}
.mask-box {
  position: fixed;
  top: 0;
  left: 0;
  z-index: 9999;
  width: 100%;
  height: 100%;
  @include center;
}
@media screen and (max-width: 750px) {
  .news-box-2 {
    display: flex;
    flex-wrap: wrap;
    gap: vw(32);
  }
  .article {
    line-height: vw(38);
    padding-bottom: vw(32);
    border-bottom: vw(2) solid #ececee;
    min-height: calc(100vh - vw(304));
  }
  .article-title {
    font-size: vw(40);
    line-height: vw(56);
    margin-bottom: vw(32);
  }
  .article-desc {
    margin-bottom: vw(48);
  }
}
</style>
