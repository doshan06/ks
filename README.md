
readme = """# 即时天气 - 实时天气预报网页应用

一个简洁美观的天气预报网页应用，支持城市搜索、实时天气展示、未来5天预报、地理位置定位和数据持久化功能。

## 在线体验

直接在浏览器中打开 `weather_app.html` 即可使用，无需安装任何依赖。

## 功能特性

| 功能 | 说明 |
|------|------|
| 城市搜索 | 输入城市名称，支持回车键和按钮触发 |
| 实时天气 | 温度、天气状况、湿度、风速、能见度 |
| 未来预报 | 未来5天最高/最低温度及天气状况 |
| 地理定位 | 一键获取当前位置天气 |
| 历史记录 | localStorage 保存最近搜索的城市 |

## 技术栈

- HTML5 - 语义化页面结构
- CSS3 - 响应式布局、Flexbox/Grid、动画效果
- JavaScript (ES6+) - 现代语法：async/await、箭头函数、解构赋值
- Fetch API - 网络请求
- Open-Meteo API - 免费天气数据源（无需 API Key）
- BigDataCloud API - 反向地理编码（经纬度转城市名）

---

## 关键技术详解

### 1. 页面结构与语义化 HTML

使用语义化标签构建清晰的文档结构：

```html
<nav class="navbar">      <!-- 顶部导航 -->
<section class="search-section">  <!-- 搜索区域 -->
<main class="main-content">     <!-- 主内容区 -->
```

**对应代码位置**：weather_app.html 第 1-120 行

---

### 2. 响应式 CSS 布局

使用 CSS 变量统一管理主题色，Flexbox + Grid 实现自适应布局：

```css
:root {
    --primary: #2563EB;
    --bg: #F8FAFC;
    --card-bg: #FFFFFF;
    --shadow: 0 1px 3px rgba(0,0,0,0.08);
}

/* 搜索框 Flex 布局 */
.search-box {
    display: inline-flex;
    align-items: center;
}

/* 天气详情 Grid 布局 */
.weather-details {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
}

/* 移动端适配 */
@media (max-width: 600px) { ... }
```

**对应代码位置**：weather_app.html 第 15-300 行

---

### 3. 异步数据获取（async/await + Fetch API）

使用现代 JavaScript 异步语法链式调用多个 API：

```javascript
// 地理编码：城市名转经纬度
async function geocodeCity(city) {
    let url = `https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(city)}&count=10&language=zh&format=json`;
    let res = await fetch(url);
    let data = await res.json();
    
    // 优先返回中国城市
    if (data.results && data.results.length > 0) {
        const cnResult = data.results.find(r => r.country_code === 'CN');
        if (cnResult) return cnResult;
        return data.results[0];
    }
    // 中文无结果则转英文搜索
}

// 天气数据获取
async function fetchWeather(lat, lon) {
    const url = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current=...&daily=...`;
    const res = await fetch(url);
    return res.json();
}
```

**对应代码位置**：weather_app.html 第 580-640 行

---

### 4. 城市搜索优化（中英文对照表）

Open-Meteo 对中文城市名支持有限，内置 300+ 中国城市中英文对照表实现自动转换：

```javascript
const cityNameMap = {
    '深圳': 'Shenzhen',
    '杭州': 'Hangzhou',
    '成都': 'Chengdu',
    // ... 300+ 城市
};

// 搜索逻辑：中文无结果 -> 自动转英文再搜索
async function geocodeCity(city) {
    // 1. 先中文搜索
    // 2. 无结果时：cityNameMap[city] 获取英文名
    // 3. 用英文再搜索，优先返回中国结果
}
```

**对应代码位置**：weather_app.html 第 380-570 行

---

### 5. 地理位置定位

结合浏览器 Geolocation API 和反向地理编码：

```javascript
function locateAndSearch() {
    navigator.geolocation.getCurrentPosition(
        async (position) => {
            const { latitude, longitude } = position.coords;
            // 反向地理编码：经纬度转城市名
            const cityInfo = await reverseGeocode(latitude, longitude);
            // 获取天气
            const weatherData = await fetchWeather(latitude, longitude);
            renderWeather(cityInfo, weatherData);
        },
        (err) => { /* 错误处理 */ },
        { timeout: 10000 }
    );
}

// 反向地理编码（BigDataCloud 免费 API）
async function reverseGeocode(lat, lon) {
    const url = `https://api.bigdatacloud.net/data/reverse-geocode-client?latitude=${lat}&longitude=${lon}&localityLanguage=zh`;
    const data = await (await fetch(url)).json();
    return {
        name: data.city || data.locality || '当前位置',
        // ...
    };
}
```

**对应代码位置**：weather_app.html 第 650-700 行

---

### 6. 数据持久化（localStorage）

保存用户最近搜索的 6 个城市，刷新后自动恢复：

```javascript
function getRecentSearches() {
    return JSON.parse(localStorage.getItem('weather_recent') || '[]');
}

function saveRecentSearch(city) {
    let recent = getRecentSearches();
    recent = recent.filter(c => c !== city);  // 去重
    recent.unshift(city);                      // 最新放前面
    recent = recent.slice(0, 6);             // 最多保留6个
    localStorage.setItem('weather_recent', JSON.stringify(recent));
}

// 初始化时自动加载上次搜索
const recent = getRecentSearches();
if (recent.length > 0) {
    cityInput.value = recent[0];
    searchCity(recent[0]);
}
```

**对应代码位置**：weather_app.html 第 720-760 行

---

### 7. WMO 天气代码映射

将 Open-Meteo 返回的数字天气代码转换为中文描述和 Emoji 图标：

```javascript
const weatherCodes = {
    0:  { desc: '晴朗', icon: '☀️' },
    1:  { desc: '大部晴朗', icon: '🌤' },
    2:  { desc: '多云', icon: '⛅' },
    3:  { desc: '阴天', icon: '☁️' },
    61: { desc: '小雨', icon: '🌧' },
    95: { desc: '雷雨', icon: '⛈' },
    // ... 共 30+ 种天气状况
};

function getWeatherInfo(code) {
    return weatherCodes[code] || { desc: '未知', icon: '❓' };
}
```

**对应代码位置**：weather_app.html 第 330-370 行

---

### 8. 温度条可视化

未来预报中的温度条使用相对位置计算，直观展示温差范围：

```javascript
const maxTemp = Math.max(...daily.temperature_2m_max.slice(1, 6));
const minTemp = Math.min(...daily.temperature_2m_min.slice(1, 6));
const tempRange = maxTemp - minTemp || 1;

// 计算温度条的位置和宽度
const barWidth = ((high - low) / tempRange) * 100;
const barOffset = ((low - minTemp) / tempRange) * 100;

// 渲染
<div class="temp-bar">
    <div class="temp-bar-fill" style="left: ${barOffset}%; width: ${barWidth}%"></div>
</div>
```

**对应代码位置**：weather_app.html 第 670-690 行

---

## 项目结构

```
.
├── weather_app.html    # 完整单页应用（HTML + CSS + JS）
└── README.md           # 项目说明文档
```

单文件设计，无需构建工具，直接浏览器打开即可运行。

## 数据来源

| 服务 | 用途 | 特点 |
|------|------|------|
| Open-Meteo Geocoding | 城市名转经纬度 | 免费，无需 Key |
| Open-Meteo Forecast | 天气数据 | 免费，WMO 标准代码 |
| BigDataCloud | 经纬度转城市名 | 免费客户端 API |

## 浏览器兼容性

- Chrome / Edge / Firefox / Safari 最新版
- 需支持 fetch、async/await、localStorage、navigator.geolocation

## 作者

基于 Open-Meteo 免费 API 构建
"""

# 使用 LF 换行符（Unix 格式），确保 GitHub 兼容
with open('/mnt/agents/output/README.md', 'w', encoding='utf-8', newline='\n') as f:
    f.write(readme)

print("已重新生成 README.md（Unix 换行符格式）")
