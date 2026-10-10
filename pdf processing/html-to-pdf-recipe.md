# راهنمای ساخت PDFِ زیبا و فارسی از خروجی HTML

> این روشی است که در «قانون پس‌انداز من برای زندگی» استفاده شد. همه‌ی کدها از همان فایل تست‌شده برداشته شده‌اند.

## ۱) ایده‌ی اصلی در یک جمله
PDF **اسکرین‌شاتِ صفحه‌ی وب نیست**؛ یک «طرح چاپیِ جدا» است که اندازه‌ی A4 دارد، با **همان موتور رندر مرورگر** عکس گرفته می‌شود و بعد داخل یک PDF سه‌صفحه‌ای قرار می‌گیرد.

## ۲) مکانیزم، گام‌به‌گام
1. **برگه‌های A4 جدا از UI:** سه `div.sheet` با اندازه‌ی ثابت `794×1121px` (A4 در ۹۶dpi) که از داده‌های کاربر ساخته می‌شوند و در صفحه مخفی‌اند (`visibility:hidden; height:0; overflow:hidden`، نه `left:-10000px` که در RTL اسکرول افقی می‌سازد).
2. **طراحی مخصوص چاپ:** رنگ‌ها و فونت‌ها در برگه ثابت‌اند (همیشه روشن، همیشه Estedad) تا تم تیره‌ی صفحه روی PDF اثر نگذارد. اندازه‌ها px هستند نه rem.
3. **جاسازی فونت:** فونت‌ها به‌صورت `base64` داخل `@font-face` در خود فایل‌اند؛ نه به اینترنت نیاز است و نه فونت جایگزین می‌آید.
4. **fit خودکار:** اگر محتوای یک برگه از ارتفاع A4 بلندتر شد، `transform:scale` آن را کوچک می‌کند تا همیشه در یک صفحه جا شود. صفحه‌ی اول دو ستونی (`column-count:2`) است.
5. **رندر با مرورگر، نه با کتابخانه:** برگه با `XMLSerializer` به XHTML تبدیل می‌شود، همراه `<style>` صفحه داخل یک `<svg><foreignObject>` می‌رود و به‌صورت تصویر روی `canvas` کشیده می‌شود (۲× برای وضوح). چون خود Chrome/Safari متن را می‌کشد، اتصال حروف فارسی، جهت راست‌به‌چپ، علائم نگارشی، گرادیان و سایه دقیقاً مثل صفحه‌ی وب است.
6. **ساخت PDF بدون کتابخانه:** هر canvas به JPEG تبدیل و در یک نویسنده‌ی PDF کوچک (حدود ۲۰ خط) صفحه‌ی A4 می‌شود. خروجی یک `Blob` است.
7. **تحویل:** دانلود با `<a download>`؛ اشتراک با `navigator.share({files:[...]})`؛ و اگر مرورگر پشتیبانی نکرد، دانلود. دکمه‌ی «چاپ / ذخیره PDF» (`window.print()` با `@page{size:A4;margin:0}`) هم به‌عنوان مسیر پشتیبان و برای متن قابل‌انتخاب هست.

## ۳) چرا html2pdf / html2canvas را کنار گذاشتم
در تست، html2canvas علامت‌های نگارشی فارسی را جابه‌جا می‌کرد («،» و «:» قبل از کلمه می‌آمدند) و خروجی `html2pdf` روی عنصر مخفی یک صفحه‌ی خالی بود. رندر با خود مرورگر هر دو مشکل را حل کرد.

## ۴) کد اصلی (تست‌شده)

```js
// رندر هر برگه با خود مرورگر (SVG foreignObject) تا متن فارسی راست‌به‌چپ و علائم نگارشی دقیق بماند
async function sheetJpeg(el){
 const W=794,H=1121,S=2;
 const css=[...document.querySelectorAll('style')].map(x=>x.textContent).join('\n');
 const html=new XMLSerializer().serializeToString(el);
 const svg='<svg xmlns="http://www.w3.org/2000/svg" width="'+W*S+'" height="'+H*S+'" viewBox="0 0 '+W+' '+H+'"><foreignObject x="0" y="0" width="'+W+'" height="'+H+'"><div xmlns="http://www.w3.org/1999/xhtml" dir="rtl" lang="fa" style="direction:rtl;width:'+W+'px;height:'+H+'px"><style>/*<![CDATA[*/'+css+'/*]]>*/</style>'+html+'</div></foreignObject></svg>';
 const img=new Image();
 await new Promise((ok,no)=>{img.onload=ok;img.onerror=()=>no(new Error('svg'));img.src='data:image/svg+xml;charset=utf-8,'+encodeURIComponent(svg)});
 const cv=document.createElement('canvas');cv.width=W*S;cv.height=H*S;const c=cv.getContext('2d');
 c.fillStyle='#fff';c.fillRect(0,0,cv.width,cv.height);
 c.drawImage(img,0,0);await new Promise(r=>setTimeout(r,250));   // دور اول: بارگذاری فونت داخل SVG
 c.fillStyle='#fff';c.fillRect(0,0,cv.width,cv.height);c.drawImage(img,0,0);
 const blob=await new Promise(r=>cv.toBlob(r,'image/jpeg',0.92));
 return{w:cv.width,h:cv.height,d:new Uint8Array(await blob.arrayBuffer())};
}
// ساخت PDF سه‌صفحه‌ای A4 بدون کتابخانه‌ی خارجی
function makePdf(pages){
 const enc=new TextEncoder(),parts=[],xref=[];let off=0;
 const push=d=>{const u=typeof d==='string'?enc.encode(d):d;parts.push(u);off+=u.length};
 const obj=(n,body)=>{xref[n]=off;push(n+' 0 obj\n'+body+'\nendobj\n')};
 push('%PDF-1.4\n');
 obj(1,'<< /Type /Catalog /Pages 2 0 R >>');
 obj(2,'<< /Type /Pages /Kids ['+pages.map((_,i)=>(3+3*i)+' 0 R').join(' ')+'] /Count '+pages.length+' >>');
 pages.forEach((j,i)=>{
  const pg=3+3*i,ct=pg+1,im=pg+2,c='q 595.28 0 0 841.89 0 0 cm /Im0 Do Q';
  obj(pg,'<< /Type /Page /Parent 2 0 R /MediaBox [0 0 595.28 841.89] /Resources << /XObject << /Im0 '+im+' 0 R >> >> /Contents '+ct+' 0 R >>');
  obj(ct,'<< /Length '+c.length+' >>\nstream\n'+c+'\nendstream');
  xref[im]=off;push(im+' 0 obj\n<< /Type /XObject /Subtype /Image /Width '+j.w+' /Height '+j.h+' /ColorSpace /DeviceRGB /BitsPerComponent 8 /Filter /DCTDecode /Length '+j.d.length+' >>\nstream\n');push(j.d);push('\nendstream\nendobj\n');
 });
 const n=3+3*pages.length,xo=off;let x='xref\n0 '+n+'\n0000000000 65535 f \n';
 for(let k=1;k<n;k++)x+=String(xref[k]).padStart(10,'0')+' 00000 n \n';
 push(x+'trailer\n<< /Size '+n+' /Root 1 0 R >>\nstartxref\n'+xo+'\n%%EOF');
 return new Blob(parts,{type:'application/pdf'});
}
async function pdfBlob(){
 await document.fonts.ready;buildSheets();
 const pages=[];for(const id of ['s1','s2','s3'])pages.push(await sheetJpeg($(id)));
 return makePdf(pages);
}
```

### تنظیم خودکار اندازه برای جاشدن در یک A4

```js
function fit(sheet){
 const inn=sheet.querySelector('.in'),avail=1121-38-60;let k=1;
 for(let i=0;i<8;i++){inn.style.width=(714/k)+'px';inn.style.transform='scale('+k+')';const vis=inn.offsetHeight*k;if(vis<=avail)break;k=k*avail/vis*.98}
}
```

### اشتراک PDF

```js
async function sharePDF(){
 if(!need())return;
 try{const b=await pdfBlob(),f=new File([b],'my-savings-rule.pdf',{type:'application/pdf'});
  if(navigator.canShare&&navigator.canShare({files:[f]}))await navigator.share({files:[f],title:'قانون پس‌انداز من',text:'قانون پس‌انداز من برای زندگی 🎯'});
  else{save(b,'my-savings-rule.pdf');toast('اشتراک مستقیم پشتیبانی نمی‌شود؛ PDF دانلود شد')}
 }catch(e){if(e.name!=='AbortError')toast('اشتراک‌گذاری انجام نشد؛ ابتدا PDF را دانلود کن')}
}
```

### CSS لازم برای برگه‌ها

```css
#sheets{position:absolute;top:0;right:0;width:794px;height:0;overflow:hidden;visibility:hidden;pointer-events:none;direction:rtl}
.sheet,.sheet *{font-family:'Estedad',Tahoma,sans-serif}      /* فونت جاسازی‌شده */
.sheet{width:794px;height:1121px;overflow:hidden;background:#fff;padding:38px 40px 60px;position:relative;color:#1e293b}
.sheet .in{width:714px;transform-origin:top right;font-size:13px;line-height:1.9}
.sheet .cols{column-count:2;column-gap:24px}                  /* صفحه‌ی متن دوستونه */
@media print{@page{size:A4;margin:0} body>*:not(#sheets){display:none!important} #sheets{position:static;width:auto;height:auto;overflow:visible;visibility:visible}}
```

## ۵) محدودیت‌ها (صادقانه)
- متن داخل PDF **عکس** است؛ قابل انتخاب و جست‌وجو نیست و حجم هر ۳ صفحه حدود ۸۰۰KB می‌شود.
- در Safari رندر `foreignObject` گاهی فونت را دیر می‌گیرد؛ به همین دلیل کد یک دور رندر آزمایشی و ۲۵۰ میلی‌ثانیه صبر دارد. روی iPhone/Safari تست نشده؛ اگر مشکل ماند، «چاپ / ذخیره PDF» را پیشنهاد بده.
- برای PDFِ متنی و قابل‌انتخاب: روی سرور با Playwright (`page.pdf({format:'A4',printBackground:true})`) یا از راه چاپ مرورگر بگیر.

## ۶) پرامپت آماده (کپی کن و به هر هوش مصنوعی بده)

```
یک فایل HTML تک‌فایلی (بدون وابستگی خارجی) بساز برای «[موضوع فرم]» به فارسی و راست‌به‌چپ.

ساختار:
- ویزارد: هر سؤال یک صفحه‌ی جدا، با پیشنهادهای کلیکی، نوار پیشرفت، دکمه‌ی «رد کردن» برای سؤال‌های اختیاری، ذخیره‌ی خودکار در localStorage و ادامه از همان‌جا بعد از رفرش.
- دکمه‌های خروجی (دانلود PDF، چاپ، اشتراک PDF، اشتراک تصویر استوری) فقط در صفحه‌ی آخر باشند.
- تم خودکار/روشن/تیره (CSS variables + prefers-color-scheme) و تنظیم اندازه و نوع فونت در یک پنل ⚙️.

PDF:
- دقیقاً [N] صفحه‌ی A4. برای هر صفحه یک div.sheet با اندازه‌ی ثابت 794×1121px بساز که از پاسخ‌های کاربر پر می‌شود، همیشه روشن است و فونت ثابت دارد. صفحه‌ی متن را دوستونه و خلاصه کن تا در یک برگه جا شود؛ اگر سرریز شد با transform:scale به‌صورت خودکار کوچک شود.
- هر برگه با XMLSerializer به XHTML تبدیل شود، همراه <style> داخل svg>foreignObject برود، روی canvas با ضریب ۲ کشیده شود (یک دور تست برای لود فونت)، به JPEG تبدیل شود و با یک نویسنده‌ی PDF کوچک (Catalog/Pages/Page/Image با DCTDecode) به PDF بدل شود. از html2canvas و html2pdf استفاده نکن، چون علائم نگارشی فارسی را خراب می‌کنند.
- فونت [نام فونت] را با base64 داخل @font-face جاسازی کن.
- اشتراک با navigator.share({files}) و در صورت عدم پشتیبانی، دانلود. یک دکمه‌ی پشتیبان «چاپ / ذخیره PDF» با @page{size:A4;margin:0} هم بگذار.
- پایین هر صفحه‌ی PDF بنویس: «تنظیم‌شده توسط [نام] · [@آیدی]» (آیدی را داخل <bdi dir="ltr"> بگذار) و شماره‌ی صفحه.

کارت استوری: canvas 1080×1920 با همان فونت و این جمله: «از تاریخ … برای هدفِ … در … پس‌انداز/سرمایه‌گذاری می‌کنم».

تست: با Playwright همه‌ی ویزارد را طی کن، PDF را بگیر، با pdfinfo تعداد صفحه را بشمار، با pdftoppm هر صفحه را به PNG تبدیل کن و ظاهر را (ترتیب راست‌به‌چپ، سرریز، رنگ) بررسی کن؛ navigator.share را mock کن و فایل ارسالی را بررسی کن.
```

## ۷) روش تست (همان که استفاده شد)
```bash
pip install playwright && playwright install chromium
python3 test.py                 # ویزارد را طی می‌کند، دانلود PDF را می‌گیرد، share را mock می‌کند
pdfinfo out.pdf                 # باید Pages: 3 باشد
pdftoppm -r 80 -png out.pdf p   # هر صفحه را به عکس ببین
```
