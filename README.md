# Node.js Google Maps Scraper：不写爬虫框架也能批量抓取地图数据的实战方案

做本地 SEO 或者帮客户跑竞品分析的时候，Google Maps 上的商家信息几乎是绑定需求——店名、评分、电话、营业时间、评论数，这些数据手动复制粘贴能让人崩溃。我去年开始用 Node.js 写自动化脚本抓 Google Maps 数据，踩了一堆坑之后才摸清楚一条相对稳定的路径。这篇文章把我实际跑通的方案、代码思路、以及怎么用 ScraperAPI 绕过反爬限制都讲清楚。

[👉 查看 ScraperAPI 全部套餐与免费试用额度](https://www.scraperapi.com/?fp_ref=coupons)

## 为什么直接用 Puppeteer 硬抓 Google Maps 迟早会翻车

Google Maps 的前端是重度 JavaScript 渲染的单页应用，数据不在初始 HTML 里，而是通过内部 RPC 接口动态加载。这意味着：

- 普通 HTTP 请求库（axios、node-fetch）拿到的是空壳页面，没有任何商家数据。
- 用 Puppeteer 或 Playwright 做无头浏览器渲染可以拿到数据，但 Google 的反爬机制会在短时间内识别自动化流量，触发验证码或直接封 IP。
- 即使加了代理池轮换，Google 对数据中心 IP 段的识别精度很高，廉价代理的存活率极低。

我自己最初的方案就是 Puppeteer + 免费代理列表，跑了不到 200 条请求就开始大面积 CAPTCHA，效率掉到几乎为零。后来切到 ScraperAPI 之后，这些问题基本一次性解决了——它在底层帮你处理代理轮换、浏览器指纹、重试逻辑和验证码绕过，你只需要发一个普通的 HTTP 请求就行。

## ScraperAPI 在 Google Maps 抓取场景下解决了什么

ScraperAPI 本质上是一个代理即服务平台，但它比单纯的代理池多做了几件关键的事：

- **自动渲染 JavaScript**：通过 `render=true` 参数，它会在服务端用真实浏览器环境渲染页面后再返回完整 HTML，省掉你本地跑无头浏览器的资源消耗。
- **智能代理轮换**：超过 4000 万个住宅和数据中心 IP 的池子，每次请求自动分配不同出口 IP，Google 很难建立行为关联。
- **内置重试与验证码处理**：遇到 CAPTCHA 或临时封锁时自动换 IP 重试，成功率官方标称在 99% 以上。
- **地理定位**：可以指定请求从哪个国家/城市发出，对抓取本地化 Google Maps 结果至关重要——比如你想抓纽约的餐厅数据，就让请求从美国 IP 出去。

对于 Node.js 开发者来说，接入方式极其简单：把目标 URL 作为参数传给 ScraperAPI 的端点，剩下的事情它全包了。

[👉 免费注册 ScraperAPI 获取 5000 次 API 调用额度](https://www.scraperapi.com/?fp_ref=coupons)

## 实战：用 Node.js + ScraperAPI 抓取 Google Maps 搜索结果

下面是一个完整的、可直接跑的 Node.js 脚本示例，抓取 Google Maps 上某个关键词的搜索结果页商家列表。

### 环境准备

```bash
mkdir google-maps-scraper && cd google-maps-scraper
npm init -y
npm install axios cheerio
```

### 核心代码

```javascript
const axios = require('axios');
const cheerio = require('cheerio');

const API_KEY = 'YOUR_SCRAPERAPI_KEY'; // 注册后在 Dashboard 获取
const QUERY = 'plumbers in Austin TX';
const ENCODED_QUERY = encodeURIComponent(QUERY);

// Google Maps 搜索结果的 URL 结构
const targetUrl = `https://www.google.com/maps/search/${ENCODED_QUERY}/`;

async function scrapeGoogleMaps() {
  try {
    const response = await axios.get('https://api.scraperapi.com', {
      params: {
        api_key: API_KEY,
        url: targetUrl,
        render: 'true',        // 启用 JS 渲染
        country_code: 'us',    // 从美国 IP 发出请求
        wait_for_selector: 'div[role="feed"]' // 等待商家列表容器加载
      },
      timeout: 12000 // Google Maps 渲染较慢，给足超时时间
    });

    const html = response.data;
    const $ = cheerio.load(html);

    const businesses = [];

    // Google Maps 搜索结果中每个商家卡片的选择器
    $('div[role="feed"] > div').each((index, element) => {
      const name = $(element).find('a[aria-label]').attr('aria-label') || '';
      const rating = $(element).find('span[role="img"]').attr('aria-label') || '';
      const address = $(element).find('[data-value="address"]').text().trim() || '';

      if (name) {
        businesses.push({
          name,
          rating,
          address,
          position: index + 1
        });
      }
    });

    console.log(`Found ${businesses.length} businesses:`);
    console.log(JSON.stringify(businesses, null, 2));
    return businesses;

  } catch (error) {
    console.error('Scraping failed:', error.message);
    if (error.response) {
      console.error('Status:', error.response.status);
    }
  }
}

scrapeGoogleMaps();
```

### 关于选择器的说明

Google Maps 的DOM 结构没有稳定的 class 命名（它用的是混淆后的随机字符串），所以上面的选择器基于 ARIA 属性和 role 属性来定位，这些相对稳定。但 Google 会不定期调整页面结构，如果脚本突然抓不到数据，优先检查选择器是否需要更新。

## 进阶：用 ScraperAPI 的 Google Maps 专用结构化端点

如果你不想自己解析 HTML，ScraperAPI 还提供了专门针对 Google 搜索和 Google Maps 的结构化数据 API（Structured Data Endpoints），直接返回 JSON 格式的干净数据。

```javascript
const axios = require('axios');

const API_KEY = 'YOUR_SCRAPERAPI_KEY';

async function scrapeWithStructuredAPI() {
  try {
    const response = await axios.get('https://api.scraperapi.com/structured/google/maps/search', {
      params: {
        api_key: API_KEY,
        query: 'coffee shops in Seattle',
        ll: '@47.6062,-122.3321,12z' // 纬度,经度,缩放级别
      },
      timeout: 6000
    });

    const results = response.data;
    console.log(`Total results: ${results.local_results?.length || 0}`);

    if (results.local_results) {
      results.local_results.forEach((biz, i) => {
        console.log(`${i + 1}. ${biz.title} | Rating: ${biz.rating} | Reviews: ${biz.reviews}`);
        console.log(`   Address: ${biz.address}`);
        console.log(`   Phone: ${biz.phone || 'N/A'}`);
        console.log('---');
      });
    }

    return results;
  } catch (error) {
    console.error('Structured API error:', error.message);
  }
}

scrapeWithStructuredAPI();
```

这个端点省掉了你写选择器和维护解析逻辑的麻烦，返回的 JSON 里直接包含商家名称、评分、评论数、地址、电话、营业时间、经纬度等字段。对于批量抓取场景，这条路径的维护成本低得多。

[👉 立即注册获取 API Key开始免费调用](https://www.scraperapi.com/?fp_ref=coupons)

## 批量抓取与并发控制

实际项目中你不会只抓一个关键词。下面是一个支持多关键词批量抓取、带并发限制的版本：

```javascript
const axios = require('axios');

const API_KEY = 'YOUR_SCRAPERAPI_KEY';
const CONCURRENCY_LIMIT = 5; // ScraperAPI 免费版并发上限为 5

const queries = [
  'dentists in Chicago',
  'auto repair in Houston',
  'yoga studios in Portland',
  'italian restaurants in San Francisco',
  'hair salons in Miami'
];

function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function scrapeQuery(query) {
  try {
    const response = await axios.get('https://api.scraperapi.com/structured/google/maps/search', {
      params: {
        api_key: API_KEY,
        query: query
      },
      timeout: 6000
    });
    return { query, results: response.data.local_results || [], success: true };
  } catch (error) {
    return { query, results: [], success: false, error: error.message };
  }
}

async function batchScrape(queries, concurrency) {
  const results = [];

  for (let i = 0; i < queries.length; i += concurrency) {
    const batch = queries.slice(i, i + concurrency);
    const batchResults = await Promise.all(batch.map(q => scrapeQuery(q)));
    results.push(...batchResults);

    // 批次间加延迟，避免触发速率限制
    if (i + concurrency < queries.length) {
      await delay(2000);
    }
  }

  // 汇总报告
  const successful = results.filter(r => r.success);
  const failed = results.filter(r => !r.success);

  console.log(`\nBatch complete: ${successful.length} succeeded, ${failed.length} failed`);
  successful.forEach(r => {
    console.log(`  "${r.query}" → ${r.results.length} businesses found`);
  });

  return results;
}

batchScrape(queries, CONCURRENCY_LIMIT);
```

## ScraperAPI 套餐怎么选

ScraperAPI 的定价按 API 调用次数计费，不同套餐的并发数和高级功能（JS 渲染、地理定位、结构化端点）有差异：

| 套餐 | 价格 | API 调用次数 | 并发数 | 适用场景 |
|------|----------|---|---|---|
| Free | $0 | 5,000 次 | 5 | 开发测试、验证可行性 |
| Hobby | $49/月 | 100,000 次 | 10 | 小规模定期抓取 |
| Startup | $149/月 | 500,000 次 | 25 | 中等规模商业项目 |
| Business | $299/月 | 1,000,000 次 | 50 | 大规模数据采集 |
| Enterprise | 定制 | 定制 | | 超大规模或特殊需求 |

几个选购建议：

- Google Maps 页面因为需要 JS 渲染，每次调用消耗的额度比普通页面多（渲染请求按 10 次计费），选套餐时要把这个倍率算进去。
- 如果你主要用结构化数据端点，每次调用按 25 次计费，但省掉了你自己维护解析器的时间成本。
- 免费的 5000 次额度足够你跑通整个流程、验证数据质量，再决定要不要付费。

[👉 查看 ScraperAPI 完整定价与功能对比](https://www.scraperapi.com/?fp_ref=coupons)

## 抓取 Google Maps 商家详情页

除了搜索结果列表，很多场景需要进入单个商家的详情页抓取更完整的信息（完整评论列表、营业时间、照片数量、网站链接等）。思路类似：

```javascript
async function scrapeBusinessDetail(placeUrl) {
  try {
    const response = await axios.get('https://api.scraperapi.com', {
      params: {
        api_key: API_KEY,
        url: placeUrl,
        render: 'true',
        wait_for_selector: 'h1' // 等待商家名称加载
      },
      timeout: 120000
    });

    const $ = cheerio.load(response.data);

    const detail = {
      name: $('h1').first().text().trim(),
      rating: $('span[aria-hidden="true"]').first().text().trim(),
      totalReviews: $('button[aria-label*="reviews"]').text().trim(),
      address: $('button[data-item-id="address"]').text().trim(),
      phone: $('button[data-item-id*="phone"]').text().trim(),
      website: $('a[data-item-id="authority"]').attr('href') || '',
      hours: []
    };

    // 提取营业时间
    $('table[aria-label*="hours"] tr').each((_, row) => {
      const day = $(row).find('td').first().text().trim();
      const time = $(row).find('td').last().text().trim();
      if (day) detail.hours.push({ day, time });
    });

    return detail;
  } catch (error) {
    console.error(`Failed to scrape ${placeUrl}:`, error.message);
    return null;
  }
}
```

## 几个实战中容易踩的坑

**超时设置要给够**

Google Maps 的 JS 渲染比普通网页慢很多，我建议 timeout 至少设 90-120 秒。ScraperAPI 在服务端渲染完成后才会返回响应，如果你客户端超时设太短，会白浪费一次 API 调用额度。

**地理定位参数很重要**

Google Maps 的搜索结果高度依赖请求者的地理位置。如果你在中国服务器上跑脚本但想抓美国本地商家数据，不加 `country_code` 参数可能会拿到完全不同的结果集，甚至被重定向到 Google 的其他区域版本。

**选择器维护是长期成本**

如果你用的是自己解析 HTML 的方案，Google Maps 的 DOM 结构大约每隔几周到几个月会有一次调整。结构化数据端点虽然单次调用成本更高，但免去了这个维护负担。对于生产环境的持续抓取任务，我建议优先用结构化端点。

**遵守 Google 的服务条款**

抓取 Google Maps 数据存在法律灰色地带。ScraperAPI 作为工具本身是合法的代理服务，但你需要自行评估抓取行为是否符合 Google 的 ToS 以及你所在地区的数据保护法规。建议只抓取公开可见的商业信息，不要大规模存储用户生成内容（如评论全文），并在合理频率内操作。

## 与其他方案的对比

| 方案 | 优点 | 缺点 |
|------|------|------| Puppeteer + 免费代理 | 零成本 | 封 IP 快、维护代理池耗精力、本地资源消耗大 |
| Puppeteer + 付费住宅代理 | 成功率尚可 | 需要自己处理指纹、重试、验证码；代理费用不低 |
| ScraperAPI（渲染模式） | 开箱即用、成功率高、无需本地浏览器 | 按调用次数付费、渲染请求消耗额度较多 |
| ScraperAPI（结构化端点） | 直接返回 JSON、零解析维护 | 单次消耗额度最高、灵活性略低 |
| Google Maps Platform API（官方） | 数据最准确、完全合规 | 价格贵（$32/1000 次 Place Details）、有使用限制 |

对于大多数中小规模的抓取需求，ScraperAPI 在成本和易用性之间的平衡点是最合理的。如果你的项目预算充足且对合规性要求极高，Google 官方的 Places API 是更稳妥的选择，但价格差距明显。

[👉 免费试用 ScraperAPI 跑通你的第一个 Google Maps 抓取脚本](https://www.scraperapi.com/?fp_ref=coupons)

## 完整项目结构建议

如果你打算把这个抓取逻辑做成一个可维护的项目，建议按以下结构组织：

```
google-maps-scraper/
├── src/
│   ├── config.js          # API key、并发数、超时等配置
│   ├── scraper.js         # 核心抓取逻辑
│   ├── parser.js          # HTML 解析逻辑（如果不用结构化端点）
│   └── batch.js           # 批量任务调度
├── data/
│   ├── queries.json       # 待抓取的关键词列表
│   └── results/           # 抓取结果输出目录
├── package.json
└── README.md
```

把配置、抓取、解析、调度分开，后续换关键词或调整并发策略时改动面最小。

## 总结

用 Node.js 抓取 Google Maps 数据，核心难点不在代码逻辑本身，而在于怎么稳定地绕过 Google 的反爬机制。ScraperAPI 把代理管理、浏览器渲染、验证码处理这些脏活累活封装成了一个 HTTP 接口，让你可以专注在数据处理和业务逻辑上。

免费版的 5000 次调用额度足够你验证整条链路是否跑得通。如果数据质量和成功率符合预期，再根据实际调用量选对应的付费套餐。

[👉 立即注册 ScraperAPI 开始你的 Google Maps 数据抓取项目](https://www.scraperapi.com/?fp_ref=coupons)
