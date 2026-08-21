<div align="center">

# Contributing · المساهمة

[🇬🇧 English](#english) · [🇸🇦 العربية](#العربية)

</div>

---

## English

Thank you for considering a contribution. This repository is an educational reference, so the bar is **accuracy and clarity**, not completeness.

### What is most welcome

| Priority | Contribution |
|:--:|---|
| 🔴 **Highest** | **Factual corrections**, especially anywhere the text over-claims about real Google internals |
| 🔴 **Highest** | **Arabic language improvements** — better terminology, more natural phrasing, typos |
| 🟠 High | Clearer diagrams, or diagrams that fix a misleading simplification |
| 🟠 High | Missing citations, or corrections to existing ones |
| 🟡 Medium | New sections covering a genuinely missing subsystem |
| 🟡 Medium | Translations into additional languages |
| 🟢 Low | Formatting, link fixes, spelling |

### The accuracy rule

This repository documents **publicly available** information only. Please do not add:

- Anything from an NDA, an internal document, or a private conversation
- Claims about current Google internals stated as fact
- Numbers presented as measured when they are estimated

If you add a number, either cite a public source or label it explicitly as an estimate. If you are correcting a number, please say where the better figure comes from.

### Style conventions

**Prose**
- Explain *why* a design choice was made, not only *what* it is. A section that lists components without naming the constraint that forced them is not finished.
- State the cost of every trade-off. "We chose X" is incomplete; "We chose X, accepting Y" is complete.
- Prefer concrete numbers over adjectives — "225 TB" beats "very large".

**Diagrams**
- Mermaid only. No image files — they cannot be diffed, translated or maintained.
- Every diagram gets the next free `D-NN` id and a caption line above it:
  ```
  > **Diagram D-79 — What it shows**
  ```
- Quote every node label: `A["Text here"]`. Unquoted labels break on punctuation.
- Use the standard colour classes (see [diagrams/README.md](diagrams/README.md)).
- Validate before opening a PR:
  ```bash
  npx -y @mermaid-js/mermaid-cli -i your-diagram.mmd -o /tmp/test.svg
  ```

**Bilingual structure**
- Every change to `docs/en/NN-*.md` should have a matching change in `docs/ar/NN-*.md`.
- If you cannot write the Arabic, open the PR anyway with the English change and say so — someone can pair with you. An English-only PR is far better than no PR.
- Arabic prose is wrapped in `<div dir="rtl" align="right">` blocks. Mermaid code fences must sit **outside** those blocks (close the div before the fence, reopen after), otherwise GitHub will not render the diagram.
- Keep the diagram `D-NN` ids identical across both languages — only the labels are translated.

### Pull request checklist

- [ ] All Mermaid diagrams render (validated locally)
- [ ] Internal links resolve
- [ ] English and Arabic are both updated, or the gap is stated in the PR description
- [ ] Any new number is cited or labelled as an estimate
- [ ] Diagram ids are sequential with no duplicates
- [ ] `diagrams/README.md` updated if diagrams were added or renamed

### Reporting an error

Open an issue with the chapter, the section number, what is wrong, and — if you have it — a public source for the correct version. Corrections are the most valuable contribution this repository can receive.

---

<div dir="rtl" align="right">

## العربية

شكرًا لتفكيرك في المساهمة. هذا المستودع مرجع تعليمي، ولذا فالمعيار هو **الدقة والوضوح** لا الاكتمال.

### أكثر المساهمات ترحيبًا

| الأولوية | المساهمة |
|:--:|---|
| 🔴 **الأعلى** | **التصويبات الواقعية**، خصوصًا حيث يبالغ النص في الادعاء بشأن بنية جوجل الداخلية |
| 🔴 **الأعلى** | **تحسينات اللغة العربية** — مصطلحات أفضل وصياغة أكثر طبيعية وتصحيح الأخطاء |
| 🟠 عالية | مخططات أوضح، أو مخططات تصحح تبسيطًا مضلِّلًا |
| 🟠 عالية | مراجع ناقصة أو تصويب المراجع الموجودة |
| 🟡 متوسطة | أقسام جديدة تغطي نظامًا فرعيًا غائبًا فعلًا |
| 🟡 متوسطة | ترجمات إلى لغات إضافية |
| 🟢 منخفضة | التنسيق وإصلاح الروابط والإملاء |

### قاعدة الدقة

يوثّق هذا المستودع المعلومات **المتاحة للعموم** فقط. فأرجو عدم إضافة:

- أي شيء من اتفاقية سرية أو وثيقة داخلية أو محادثة خاصة
- ادعاءات عن بنية جوجل الداخلية الحالية تُعرض كحقائق
- أرقام تُقدَّم كأنها مقيسة وهي مقدَّرة

فإن أضفت رقمًا، فإما أن تذكر مصدرًا عامًا أو أن تسمه صراحةً بأنه تقدير. وإن كنت تصوّب رقمًا، فاذكر من أين جاء الرقم الأفضل.

### اصطلاحات الأسلوب

**النص**
- اشرح **لماذا** اتُّخذ قرار التصميم لا **ما هو** فحسب. فالقسم الذي يعدّد المكوّنات دون تسمية القيد الذي فرضها ليس مكتملًا.
- اذكر ثمن كل مفاضلة. فعبارة «اخترنا س» ناقصة، و«اخترنا س قابلين بـص» كاملة.
- فضّل الأرقام الملموسة على الصفات — «225 تيرابايت» خير من «كبير جدًا».

**المخططات**
- Mermaid فقط. لا ملفات صور — فهي غير قابلة للمقارنة ولا للترجمة ولا للصيانة.
- كل مخطط ينال المعرّف التالي المتاح `D-NN` مع سطر تعريف فوقه.
- ضع كل تسمية عقدة بين علامتَي اقتباس: `A["النص هنا"]`، وإلا انكسرت عند علامات الترقيم.
- استخدم أصناف الألوان القياسية (راجع [diagrams/README.md](diagrams/README.md)).
- تحقق من الصحة قبل فتح طلب السحب.

**البنية ثنائية اللغة**
- كل تغيير في `docs/en/NN-*.md` ينبغي أن يقابله تغيير في `docs/ar/NN-*.md`.
- وإن لم تستطع كتابة العربية، فافتح طلب السحب بالإنجليزية وقل ذلك — فقد يشاركك أحدهم. وطلب سحب بالإنجليزية وحدها خير بكثير من لا شيء.
- النص العربي مغلَّف بكتل `<div dir="rtl" align="right">`، ويجب أن تقع أسوار شيفرة Mermaid **خارج** تلك الكتل (أغلق الكتلة قبل السور وافتحها بعده)، وإلا لم يعرض GitHub المخطط.
- أبقِ معرّفات `D-NN` متطابقة في اللغتين — فالمترجَم هو التسميات فقط.

### قائمة تحقق طلب السحب

- [ ] كل مخططات Mermaid تُعرض (تم التحقق محليًا)
- [ ] الروابط الداخلية تعمل
- [ ] الإنجليزية والعربية محدَّثتان، أو الفجوة مذكورة في وصف الطلب
- [ ] كل رقم جديد مذكور المصدر أو موسوم بأنه تقدير
- [ ] معرّفات المخططات متسلسلة بلا تكرار
- [ ] تحديث `diagrams/README.md` إن أُضيفت مخططات أو أُعيدت تسميتها

### الإبلاغ عن خطأ

افتح Issue يذكر الفصل ورقم القسم وما هو الخطأ — ومصدرًا عامًا للصيغة الصحيحة إن توفر. فالتصويبات أثمن ما يمكن أن يتلقاه هذا المستودع.

</div>

---

<div align="center">

By contributing you agree that your work is licensed under the [MIT License](LICENSE).

بالمساهمة فإنك توافق على ترخيص عملك بموجب [رخصة MIT](LICENSE).

</div>
