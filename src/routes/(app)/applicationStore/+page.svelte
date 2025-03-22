<script>
 let items = [
    { name: "ChatGPT", desc: "最强大的AI聊天机器人程序", logo: "https://www.aig123.com/wp-content/uploads/2023/06/18acd-chat.openai.com.png", link: "https://www.aig123.com/sites/65.html" },
    { name: "DeepSeek", desc: "幻方量化公司旗下的开源大模型平台", logo: "https://www.aig123.com/wp-content/uploads/2024/05/deepseek.png", link: "https://www.aig123.com/sites/2638.html" },
    { name: "GPT-4o", desc: "OpenAI 最新发布的旗舰生成式AI模型", logo: "https://t1.gstatic.cn/faviconV2?client=SOCIAL&type…ize=128&url=https://openai.com/index/hello-gpt-4o", link: "https://www.aig123.com/sites/2679.html" },
    { name: "Gemini", desc: "Google 推出的AI对话聊天机器人", logo: "https://t1.gstatic.cn/faviconV2?client=SOCIAL&type=FAVICON&fallback_opts=TYPE,SIZE,URL&size=128&url=https://gemini.google.com", link: "https://www.aig123.com/sites/185.html" },
    { name: "Perplexity", desc: "ChatGPT和谷歌结合的超级搜索引擎", logo: "https://t1.gstatic.cn/faviconV2?client=SOCIAL&type=FAVICON&fallback_opts=TYPE,SIZE,URL&size=128&url=https://www.perplexity.ai", link: "https://www.aig123.com/sites/79.html" },
    { name: "豆包", desc: "多功能AI助手，提升你的生活、学习和工作", logo: "https://lf-flow-web-cdn.doubao.com/obj/flow-doubao/doubao/web/static/image/OnboardingAvatar.bb1504ee.webp", link: "http://dis.csqixiang.cn/unpo/dbcodingaig123.html" },
    { name: "Coze", desc: "字节跳动旗下AI智能对话平台", logo: "https://www.aig123.com/wp-content/uploads/2025/02/coze.png", link: "http://dis.csqixiang.cn/unpo/cozeaig123.html" },
    { name: "AskManyAI", desc: "AI聚合平台，集成多个AI模型", logo: "https://www.aig123.com/wp-content/uploads/2024/07/askmanyai.png", link: "https://www.aig123.com/sites/3161.html" },
    { name: "Meta AI", desc: "Meta推出的AI智能助理", logo: "https://www.aig123.com/wp-content/uploads/2024/04/9aa95-www.meta.ai.png", link: "https://www.meta.ai/" },
    { name: "Poe", desc: "Quora旗下的对话机器人聚合工具", logo: "https://t1.gstatic.cn/faviconV2?client=SOCIAL&type=FAVICON&fallback_opts=TYPE,SIZE,URL&size=128&url=https://poe.com", link: "https://poe.com/" },
    { name: "Claude", desc: "Anthropic发布的ChatGPT竞争对手", logo: "https://t1.gstatic.cn/faviconV2?client=SOCIAL&type=FAVICON&fallback_opts=TYPE,SIZE,URL&size=128&url=https://claude.ai", link: "https://www.anthropic.com/" },
    { name: "扯子Coze", desc: "字节跳动旗下AI智能对话平台", logo: "https://www.aig123.com/wp-content/uploads/2025/02/coze.png", link: "https://coze.com/" }
  ];
</script>

<style>
  .container {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 20px;
    padding: 20px;
  }
  .card {
    display:flex;
    background: white;
    padding: 15px;
    border-radius: 8px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
    text-align: center;
    height: 90px;
    text-decoration: none;
    color: inherit;
  }
  .card img {
    width: 50px;
    height: 50px;
    margin-bottom: 10px;
  }
  .card h3 {
    text-align: left;
    font-size: 16px;
    margin: 5px 0;
  }
  .card p {
    display: -webkit-box; /* 必须配合 */
    -webkit-box-orient: vertical; /* 纵向排列 */
    -webkit-line-clamp: 1; /* 限制最多显示 1 行 */
    overflow: hidden; /* 超出部分隐藏 */
    text-overflow: ellipsis; /* 省略号 */
    font-size: 14px;
    color: #666;
    text-align: left;
  }
  .card .card-info{
    flex:1;
    padding-left:10px;
  }
</style>
<div class="h-screen max-h-[100dvh] transition-width duration-200 ease-in-out md:max-w-[calc(100%-260px)] w-full max-w-full flex flex-col">
  <div class="container">
    {#each items as item}
      <a href={item.link} target="_blank" class="card">
        <img src={item.logo} alt={item.name} />
        <div class="card-info">
          <h3>{item.name}</h3>
          <p>{item.desc}</p>
        </div>
      </a>
    {/each}
  </div>
</div>
