<script>
  let items = [
    { name: "秘塔AI搜索", desc: "秘塔科技推出的AI搜索引擎", logo: "https://img.alicdn.com/imgextra/i4/O1CN01EfJVFQ1uZPd7W4W6i_!!6000000006051-2-tps-112-112.png", link:'https://tongyi.aliyun.com/qianwen/'},
    { name: "Dexa AI", desc: "AI情报搜索工具", logo: "https://assets.dexa.ai/image/upload/w_48,c_thumb,g_center,q_auto:good,f_auto,dpr_2,d_entities:placeholders:assistant_ikq3b7.png,r_12,ar_1:1/entities/assistants/assistant_1", link:'https://dexa.ai/?utm_source=huyaai.cn' },
    { name: "心选", desc: "阿里旗下推出的AI搜索助手", logo: "https://img.alicdn.com/imgextra/i3/O1CN01CGhH8n1gUShGkExSm_!!6000000004145-2-tps-96-156.png", link: "https://iflow.cn/?utm_source=huyaai.cn" },
    { name: "知乎直答", desc: "知乎推出的AI搜索引擎", logo: "https://picx.zhimg.com/v2-253edf392fe4507700601efc1b5b568f.png", link: "https://zhida.zhihu.com/?utm_source=huyaai.cn" },
    { name: "AlphaSense", desc: "专为金融专业人士设计的AI搜索", logo: "https://www.alpha-sense.com/wp-content/uploads/2022/09/Logo.svg", link: "https://www.alpha-sense.com/?utm_source=huyaai.cn" },
    { name: "Miku", desc: "快速精准的AI搜索引擎", logo: "https://www.hellomiku.com/img/logo.ca75c3ea.png", link: "https://www.hellomiku.com/index?utm_source=huyaai.cn" }
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
