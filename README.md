<!doctype html>
<html lang="ar">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Stop-Loss Prototype (Solana) - كود واحد</title>
  <style>
    body { font-family: system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial; padding: 20px; direction: rtl; }
    .card { border:1px solid #e6e6e6; padding:16px; border-radius:8px; max-width:820px; margin:12px 0; }
    input, select { padding:8px; margin:6px 0; width:100%; box-sizing:border-box }
    button{ padding:10px 14px; cursor:pointer }
    .small { font-size:0.9em; color:#666 }
    table{ width:100%; border-collapse:collapse }
    td, th{ border:1px solid #eee; padding:8px }
    .danger{ color:#b30000 }
  </style>
</head>
<body>
  <h1>نموذج Stop-Loss (واجهة واحدة) — Solana</h1>
  <div class="card">
    <div id="wallet-area">
      <div><strong>الحالة:</strong> <span id="status">غير متصل</span></div>
      <button id="connect">اتصل بمحفظة Phantom</button>
      <div class="small">لن نطلب أو نخزن مفاتيحك الخاصة أبداً. كل شيء يتم محلياً في متصفحك.</div>
    </div>
  </div>

  <div class="card">
    <h3>إنشاء أمر Stop-Loss</h3>
    <label>Mint address (عقد التوكن على شبكة Solana)</label>
    <input id="mint" placeholder="مثال: So11111111111111111111111111111111111111112" />

    <label>سعر وقف الخسارة بالدولار (مثال: 0.0001)</label>
    <input id="stopPrice" type="number" step="any" placeholder="0.0001" />

    <label>الكمية (كم عدد التوكن)</label>
    <input id="amount" type="number" step="any" placeholder="100" />

    <label>بيع كم بالمئة عند التفعيل؟</label>
    <select id="sellPct">
      <option value="100">100%</option>
      <option value="50">50%</option>
      <option value="25">25%</option>
    </select>

    <button id="createOrder">أنشئ الأمر</button>
    <div id="createMsg" class="small"></div>
  </div>

  <div class="card">
    <h3>الأوامر المسجلة (محلياً)</h3>
    <table>
      <thead><tr><th>id</th><th>mint</th><th>stop USD</th><th>amt</th><th>pct</th><th>status</th><th>action</th></tr></thead>
      <tbody id="ordersTable"></tbody>
    </table>
  </div>

  <div class="card">
    <h3>سجل الأحداث</h3>
    <div id="log" style="height:160px; overflow:auto; background:#fafafa; padding:8px"></div>
  </div>

  <script>
    // === ملاحظات مهمة =========
    // هذا ملف "كود واحد" (Frontend-only) لعرض سير العمل: ربط المحفظة، إنشاء أوامر Stop-Loss محلياً، ومراقبة السعر.
    // لا يقوم هذا الكود بتنفيذ عمليات بيع حقيقية تلقائياً — عند التفعيل يظهر زر يسمح بإنشاء/طلب معاملة بيع
    // ويفترَض ربطه بخدمة مباراة (مثل Jupiter API) لإنشاء معاملة swap قابلة للتوقيع.
    // لا تطلب المفاتيح الخاصة أبداً — التوقيع يتم على جهاز المستخدم عبر Phantom.
    // ===========================

    const STATUS = document.getElementById('status');
    const connectBtn = document.getElementById('connect');
    const createBtn = document.getElementById('createOrder');
    const mintInput = document.getElementById('mint');
    const stopInput = document.getElementById('stopPrice');
    const amountInput = document.getElementById('amount');
    const pctInput = document.getElementById('sellPct');
    const ordersTable = document.getElementById('ordersTable');
    const logDiv = document.getElementById('log');

    let wallet = null; // window.solana (Phantom)
    let publicKeyBase58 = null;

    function log(msg){
      const d = new Date().toLocaleTimeString('ar-EG');
      logDiv.innerHTML = `<div>[${d}] ${msg}</div>` + logDiv.innerHTML;
    }

    // محلية: احفظ الأوامر في localStorage
    function loadOrders(){
      try{
        return JSON.parse(localStorage.getItem('stoploss_orders')||'[]');
      }catch(e){ return []; }
    }
    function saveOrders(arr){ localStorage.setItem('stoploss_orders', JSON.stringify(arr)); }

    function renderOrders(){
      const arr = loadOrders();
      ordersTable.innerHTML = '';
      for(const o of arr){
        const tr = document.createElement('tr');
        tr.innerHTML = `
          <td>${o.id}</td>
          <td style="max-width:220px; word-break:break-all">${o.mint}</td>
          <td>${o.stopPrice}</td>
          <td>${o.amount}</td>
          <td>${o.sellPct}%</td>
          <td>${o.status}</td>
          <td>
            ${o.status==='active'? `<button data-id="${o.id}" class="triggerBtn">تحقق الآن</button>`: ''}
            <button data-id="${o.id}" class="delBtn">حذف</button>
          </td>
        `;
        ordersTable.appendChild(tr);
      }
      // add listeners
      document.querySelectorAll('.delBtn').forEach(b=>b.onclick = (ev)=>{
        const id = ev.target.getAttribute('data-id');
        const arr = loadOrders().filter(x=>x.id!==id); saveOrders(arr); renderOrders(); log('حُذف الأمر '+id);
      });
      document.querySelectorAll('.triggerBtn').forEach(b=>b.onclick = async (ev)=>{
        const id = ev.target.getAttribute('data-id');
        await checkAndHandleOrder(id, true);
      });
    }

    // Connect Phantom
    async function connectWallet(){
      if (window.solana && window.solana.isPhantom){
        try{
          const resp = await window.solana.connect();
          publicKeyBase58 = resp.publicKey.toString();
          STATUS.innerText = 'متصل: ' + publicKeyBase58;
          wallet = window.solana;
          log('تم الاتصال بالمحفظة ' + publicKeyBase58);
        }catch(e){ log('إلغاء الاتصال'); }
      }else{
        alert('يرجى تثبيت محفظة Phantom، أو استخدام متصفح يدعم محفظة Solana.');
      }
    }
    connectBtn.onclick = connectWallet;

    // Create order
    createBtn.onclick = ()=>{
      const mint = mintInput.value.trim();
      const stopPrice = parseFloat(stopInput.value);
      const amount = parseFloat(amountInput.value);
      const sellPct = parseInt(pctInput.value,10);
      if(!mint || !stopPrice || !amount){ document.getElementById('createMsg').innerText = 'اكتب كل الحقول'; return; }
      const orders = loadOrders();
      const id = 'o'+Date.now();
      const o = { id, mint, stopPrice, amount, sellPct, status:'active', createdAt: new Date().toISOString() };
      orders.push(o); saveOrders(orders); renderOrders(); document.getElementById('createMsg').innerText = 'تم الإنشاء'; log('أنشئ أمر '+id+' mint:'+mint+' stop:'+stopPrice);
    };

    // Price fetcher (Dexscreener public API). NOTE: Dexscreener may not list every SPL token. For reliability use Pyth/Jupiter in production.
    async function fetchPriceUSD(mint){
      try{
        // Dexscreener API expects chain and token. For Solana tokens the API path can vary. We'll try by mint.
        const url = `https://api.dexscreener.com/latest/dex/tokens/${mint}`;
        const r = await fetch(url);
        if(!r.ok) return null;
        const j = await r.json();
        // structure may include pairs[0].priceUsd
        if(j && j.pairs && j.pairs.length>0 && j.pairs[0].priceUsd) return parseFloat(j.pairs[0].priceUsd);
        // fallback: if j.priceUsd
        if(j.priceUsd) return parseFloat(j.priceUsd);
        return null;
      }catch(e){ console.error(e); return null; }
    }

    // Check a single order and handle trigger
    async function checkAndHandleOrder(id, manual=false){
      const orders = loadOrders();
      const o = orders.find(x=>x.id===id);
      if(!o) return;
      log('جلب سعر لـ '+o.mint+' ...');
      const price = await fetchPriceUSD(o.mint);
      if(price===null){ log('تعذر جلب السعر لـ '+o.mint); if(manual) alert('تعذر جلب السعر — تحقق من mint أو استخدم خدمة سعر مختلفة.'); return; }
      log('السعر الحالي USD = '+price);
      if(price <= o.stopPrice){
        log('تم تفعيل أمر '+id+' — السعر وصل أو أقل من الهدف');
        // نعرض زر تنفيذ للمستخدم. في الواقع، هنا يجب بناء معاملة swap عبر Jupiter وطلب توقيع المستخدم.
        // سنعرض نافذة تطلب من المستخدم تنفيذ البيع يدوياً (زر Sign & Send)
        const doSell = confirm(`تم تفعيل وقف الخسارة للأمر ${id} (السعر ${price} ≤ ${o.stopPrice}). اضغط OK لتحضير المعاملة وطلب توقيعك (مبدئي).`);
        if(doSell){
          // === هنا يجب استخدام Jupiter API أو SDK لإنشاء معاملة swap جاهزة للتوقيع ===
          // مثال (تعليمات):
          // 1) اتصل بـ Jupiter API (https://quote-api.jup.ag/v1/quote) للحصول على مسار swap من mint → USDC
          // 2) استخدم endpoint createSwapTransaction من Jupiter لتوليد serialized transaction
          // 3) اطلب من المستخدم توقيع المعاملة عبر `wallet.signTransaction(tx)` ثم `connection.sendRawTransaction(...)`

          alert('تنبيه: هذا مثال توضيحي فقط. الرجاء ربط الجزء الخاص بـ Jupiter لإنشاء معاملة SWAP حقيقية.');
          // نعلّم الحالة كـ triggered محلياً
          o.status = 'triggered'; saveOrders(orders); renderOrders();
        }
      } else {
        log('لم يتم التفعيل — السعر أعلى من الهدف');
      }
    }

    // Periodic worker (runs in browser)
    async function workerTick(){
      const orders = loadOrders();
      for(const o of orders){
        if(o.status !== 'active') continue;
        try{
          const price = await fetchPriceUSD(o.mint);
          if(price !== null && price <= o.stopPrice){
            log('تم تفعيل أمر (worker) '+o.id+' — السعر='+price);
            // نعلم المستخدم بالإشعار (إذا دعم المتصفح) أو نترك علامة في الواجهة
            o.status = 'triggered';
            saveOrders(orders); renderOrders();
            // نعرض إشعار: المستخدم يحتاج لتأكيد التنفيذ.
            if(Notification && Notification.permission === 'granted'){
              new Notification('Stop-Loss triggered', { body: `الأمر ${o.id} وصل للسعر (${price} ≤ ${o.stopPrice})` });
            }
          }
        }catch(e){ console.error(e); }
      }
    }

    // Start periodic checks every 60s
    setInterval(workerTick, 60_000);

    // On load
    (function(){ renderOrders(); if(window.Notification && Notification.permission !== 'granted') Notification.requestPermission();
      if(window.solana && window.solana.isPhantom){ STATUS.innerText = 'Phantom موجود — اضغط اتصل'; }
    })();

    // Expose manual check for all
    window.checkAllNow = async ()=>{
      const orders = loadOrders();
      for(const o of orders) if(o.status==='active') await checkAndHandleOrder(o.id, true);
    };

  </script>

  <hr />
  <div class="small">ملاحظة مطوّرة: لتحويل هذا النموذج لبروتوكول حقيقي تحتاج:
    <ol>
      <li>خدمة backend لإنشاء واستضافة swap transactions عبر Jupiter SDK أو API.</li>
      <li>استعمال Pyth أو Oracle موثوق لسعر on-chain إذا كنت تريد تنفيذ on-chain آلي.</li>
      <li>في حال Auto-Execute: كتابة Solana Program (Anchor/Rust) وإيداع التوكن للعقد بعقد إسكرو مع تدقيق أمني.</li>
    </ol>
  </div>
</body>
</html>
