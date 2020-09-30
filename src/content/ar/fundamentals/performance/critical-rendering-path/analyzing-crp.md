project_path: /web/_project.yaml
book_path: /web/fundamentals/_book.yaml
description: يتطلب التعرف على نقاط التكدس في أداء مسار العرض الحرج وحلها معرفة جيدة بالمخاطر الشائعة. يمكننا الآن تنفيذ جولة عملية واستخلاص أنماط الأداء الشائعة التي تساعدك في تحسين صفحاتك.

{# wf_updated_on: 2014-04-27 #}
{# wf_published_on: 2014-03-31 #}

# تحليل أداء مسار العرض الحرج {: .page-title }

{% include "web/_shared/contributors/ilyagrigorik.html" %}


يتطلب التعرف على نقاط التكدس في أداء مسار العرض الحرج وحلها معرفة جيدة بالمخاطر الشائعة. يمكننا الآن تنفيذ جولة عملية واستخلاص أنماط الأداء الشائعة التي تساعدك في تحسين صفحاتك.



الهدف من تحسين مسار العرض الحرج هو السماح للمتصفح برسم الصفحة في أسرع وقت ممكن: ذلك أن الصفحات الأسرع تترجم إلى نسبة تفاعل أعلى وأرقام صفحات مشاهدة و[نسبة تحول محسنة](http://www.google.com/think/multiscreen/success.html). ونتيجة لذلك، نريد الحد من مقدار الوقت الذي يحتاج إليه الزائر للبدء في شاشة فارغة من خلال تحسين الموارد التي يتم تحميلها وترتيب ذلك.

وللمساعدة في توضيح هذه العملية، يمكننا البدء بأبسط حالة ممكنة والمتابعة بتصميم الصفحة لتضمين المزيد من الموارد والأنماط ومنطق التطبيق - وخلال العملية سنشاهد كذلك أشياء يمكن ألا تسير على ما يرام، وكيفية تحسين كل حالة من هذه الحالات.

وفي النهاية، يجب التنبيه على شيء قبل البدء... حتى الآن كان التركيز فقط على ما يحدث في المتصفح بعد أن يتوفر المورد (سواء أكان ملف CSS أو جافا سكريبت أو HTML) للمعالجة وتجاهلنا الوقت اللازم لجلبه سواء من ذاكرة التخزين المؤقت أو من الشبكة. سنتناول بالتفصيل كيفية تحسين عناصر الشبكات في التطبيق خلال الدرس التالي، ولكن في الوقت الحالي (وحتى تكون الأمور أكثر واقعية) سنفترض ما يلي:

* زمن انتقال الجولة (وقت استجابة النشر) على الخادم سيستغرق 100 ملي ثانية
* سيكون وقت استجابة الخادم 100 ملي ثانية لمستند HTML و10 ملي ثانية لجميع الملفات الأخرى

## تجربة Hello World

<pre class="prettyprint">
{% includecode content_path="web/fundamentals/performance/critical-rendering-path/_code/basic_dom_nostyle.html" region_tag="full" adjust_indentation="auto" %}
</pre>

سنبدأ بترميز HTML الأساسي وصورة واحدة - بدون استخدام CSS أو جافا سكريبت - حتى يكون الأمر أكثر بساطة. والآن يمكننا فتح مخطط زمني للشبكة في Chrome DevTools وفحص تدفق الموارد الناتجة:

<img src="images/waterfall-dom.png" class="center" alt="CRP">

وفقًا للمتوقع، يستغرق تنزيل ملف HTML 200 ملي ثانية. لاحظ أن الجزء الشفاف من الخط الأزرق يشير إلى الوقت الذي ينتظره المتصفح على الشبكة - أي أنه لم يتم بعد تلقي أية وحدات بايت من الاستجابة - بينما يشير الجزء الثابت إلى الوقت المطلوب لاكتمال التنزيل بعد تلقي أول وحدات بايت من الاستجابة. في المثال الموضح أعلاه، يعد تنزيل HTML ضئيلاً (<4 ك)، ولذلك لن نحتاج إلا إلى انتقال جولة لجلب الملف الكامل. ونتيجة لذلك، يستغرق جلب مستند HTML مدة 200 ملي ثانية، بحيث يتم قضاء نصف الوقت في الانتظار على الشبكة والنصف الآخر في استجابة الخادم.

وبعد توفر محتوى HTML، يكون على المتصفح تحليل وحدات بايت، وتحويلها إلى رموز مميزة، وتصميم شجرة DOM. لاحظ أن DevTools تعرض وقت حدث DOMContentLoaded على نحو سهل في الأسفل (216 ملي ثانية)، مع الاستجابة أيضًا للسطر العمودي الأزرق. تمثل الفجوة بين نهاية تنزيل HTML والسطر العمودي الأزرق (DOMContentLoaded) الوقت الذي يستغرقه المتصفح في تصميم شجرة DOM، في هذه الحالة مجرد عدد بسيط من الميلي ثانية.

وأخيرًا، هناك ملاحظة مثيرة: `الصورة الرائعة` الخاصة بنا لم تحظر حدث domContentLoaded. اتضح لنا في النهاية أنه يمكننا إنشاء شجرة العرض ورسم الصفحة بدون انتظار كل أصل من الأصول على الصفحة: **لا تعد جميع الموارد حرجة لعرض الرسم الأول والسريع**. في الحقيقة، وكما سنرى، عند التحدث عن مسار العرض الحرج، سنجد أننا نتحدث عادة عن ترميز HTML وCSS وجافا سكريبت. لا تحظر الصور العرض المبدئي للصفحة - ولكننا بالطبع يجب أن نحاول التأكد من الحصول على رسم للصور في أقرب وقت ممكن أيضًا.

ومع ذلك، يتم حظر الحدث `load` (يُعرف أيضًا باسم `onload`) على الصورة: تشير DevTools إلى الحدث أثناء التحميل عند 335 ملي ثانية. تذكر أن حدث onload يشير إلى النقطة عندما تم تنزيل ومعالجة **جميع الموارد** المطلوبة في الصفحة - وهذه النقطة تكون عند توقف أداة الانتقاء في التحميل عن الانتقاء في المتصفح وتتم الإشارة إلى هذه النقطة بسطر عمودي أحمر في التدفق.


## إضافة جافا سكريبت وCSS إلى المزيج

قد تبدو صفحة `تجربة Hello World` بسيطة على السطح، ولكن هناك الكثير من التفاصيل التي تؤدي إلى ظهور النتيجة في النهاية. ولذلك سنحتاج عمليًا إلى أكثر من مجرد HTML: ربما نحتاج إلى ورقة أنماط CSS ونص برمجي واحد أو أكثر لإضافة بعض التفاعل على الصفحة. والآن يمكننا إضافتهما إلى المزيج لنرى ما سيحدث:

<pre class="prettyprint">
{% includecode content_path="web/fundamentals/performance/critical-rendering-path/_code/measure_crp_timing.html" region_tag="full" adjust_indentation="auto" %}
</pre>

_قبل إضافة جافا سكريبت و CSS:_

<img src="images/waterfall-dom.png" alt="DOM CRP" class="center">

_مع جافا سكريبت وCSS:_

<img src="images/waterfall-dom-css-js.png" alt="DOM وCSSOM وجافا سكريبت" class="center">

لقد أدت إضافة ملفات CSS وجافا سكريبت خارجية إلى إضافة طلبين آخرين إلى التدفق، وقد تم نشر كل ذلك في وقت متساوٍ تقريبًا بواسطة المتصفح - وحتى الآن كل شيء يسير على ما يُرام. ولكن، **لاحظ أن هناك الآن اختلافًا أصغر كثيرًا في التوقيت بين domContentLoaded وأحداث onload. فماذا حدث؟**

* على العكس من مثال HTML الصريح، نحتاج الآن أيضًا إلى جلب ملف CSS وتحليله لإنشاء CSSOM، ونحن نعرف أننا بحاجة إلى كل من DOM وCSSOM لتصميم شجرة العرض.
* ونظرًا لأن لدينا أيضًا ملف جافا سكريبت لحظر المحلل على الصفحة، فقد تم حظر حدث domContentLoaded حتى يتم تنزيل ملف CSS وتحليله: وقد يستعلم جافا سكريبت عن CSSOM، ولذلك يجب حظر CSS وانتظاره حتى نتمكن من تنفيذ جافا سكريبت.

**ماذا لو تم استبدال النص البرمجي الخارجي بنص برمجي مضمَّن؟** رغم أن السؤال قد يبدو سطحيًا وعديم الفائدة، من الصعب الإجابة عنه إلى حد كبير. لقد اتضح لنا أنه حتى إذا كان النص البرمجي مضمَّنًا مباشرة في الصفحة، الوسيلة الوحيدة التي يمكن للمتصفح من خلالها معرفة ما يعتزم النص البرمجي فعله هي تنفيذه، وكما نعرف جميعًا، لا يمكننا إجراء ذلك حتى يتم إنشاء CSSOM.  باختصار، يعتبر جافا سكريبت المضمَّن أيضًا أداة حظر للمحلل.

مع ذلك وبالرغم من الحظر على CSS، هل سيؤدي تضمين النص البرمجي إلى تسريع عرض الصفحة؟ إذا كان السيناريو الأخير صعبًا، فهذا السيناريو أصعب. ويمكننا تجربته لمعرفة ما سيحدث...

_جافا سكريبت خارجي:_

<img src="images/waterfall-dom-css-js.png" alt="DOM وCSSOM وجافا سكريبت" class="center">

_جافا سكريبت مضمنة:_

<img src="images/waterfall-dom-css-js-inline.png" alt="DOM وCSSOM جافا سكريبت المضمَّن" class="center">

نحن بصدد إجراء طلب أقل، إلا أن مرات onload وdomContentLoaded هي نفسها، فلماذا؟ إننا نعرف أنه لا يهم ما إذا كان جافا سكريبت مضمَّنًا أم خارجيًا، نظرًا لأنه بمجرد وصول المتصفح إلى علامة النص البرمجي، يتم الحظر والانتظار حتى يتم إنشاء CSSOM. علاوة على ذلك، في المثال الأول، يتم تنزيل كل من CSS وجافا سكريبت بالتوازي بواسطة المتصفح ويتم الانتهاء في الوقت نفسه تقريبًا. ونتيجة لذلك، وفي هذه الحالة تحديدًا، لا يعد تضمين شفرة جافا سكريبت أمرًا مفيدًا إلى حد كبير. ولذلك هل سنقف مكتوفي الأيدي وهل انتهت بنا السبل ولن نتمكن من عرض الصفحة على نحو أسرع؟ في الواقع، هناك عدة إستراتيجيات مختلفة.

أولاً، تذكر أن جميع النصوص البرمجية المضمَّنة تؤدي إلى حظر المحلل، ولكن بالنسبة إلى النصوص البرمجية الخارجية، يمكننا إضافة الكلمة الرئيسية `async` لإلغاء حظر المحلل. سنتراجع الآن عن التضمين ونجرب ذلك:

<pre class="prettyprint">
{% includecode content_path="web/fundamentals/performance/critical-rendering-path/_code/measure_crp_async.html" region_tag="full" adjust_indentation="auto" %}
</pre>

_جافا سكريبت (الخارجي) لحظر المحلل:_

<img src="images/waterfall-dom-css-js.png" alt="DOM وCSSOM وجافا سكريبت" class="center">

_جافا سكريبت (الخارجي) غير المتزامن:_

<img src="images/waterfall-dom-css-js-async.png" alt="DOM وCSSOM وجافا سكريبت غير المتزامن" class="center">

هذا أفضل بكثير. يتم تشغيل حدث domContentLoaded بعد وقت قصير من تحليل HTML: ذلك أن المتصفح لا يعرف الحظر على جافا سكريبت ونظرًا لأنه ليست هناك نصوص برمجية أخرى لحظر المحلل، فيمكن أن يواصل إنشاء CSSOM ذلك أيضًا بالتوازي.

بدلاً من ذلك، فقد تمكنا من تجربة طريقة أخرى وضمَّنا كلاً من CSS وجافا سكريبت:

<pre class="prettyprint">
{% includecode content_path="web/fundamentals/performance/critical-rendering-path/_code/measure_crp_inlined.html" region_tag="full" adjust_indentation="auto" %}
</pre>

<img src="images/waterfall-dom-css-inline-js-inline.png" alt="DOM وتضمين CSS وتضمين جافا سكريبت" class="center">

لاحظ أن وقت _domContentLoaded_ time يعد مماثلاً للوقت الوارد في المثال السابق: بدلاً من الإشارة إلى أن جافا سكريبت غير متزامن، فقد ضمَّنا كلاً من CSS وجافا سكريبت في الصفحة نفسها. وقد أدى هذا إلى جعل صفحة HTML أكبر بكثير، ولكن الجيد في الأمر أن المتصفح لن يضطر إلى الانتظار حتى جلب أية موارد خارجية - لأن كل شيء يتوفر في الصفحة.

وكما ترى، حتى مع الصفحة البسيطة جدًا، لا يعد تحسين مسار العرض الحرج أمرًا ثانويًا: ذلك أننا بحاجة إلى استيعاب الرسم البياني للعلاقة بين الموارد المختلفة، كما نحتاج إلى تحديد الموارد `الحرجة` علاوة على الحاجة إلى الاختيار بين الإستراتيجيات المختلفة لكيفية تضمين هذه الموارد في الصفحة. لا يتوفر حل واحد لهذه المشكلة، لأن كل صفحة تختلف عن الأخرى وسيتعين عليك متابعة عملية شبيهة بنفسك لضبط الإستراتيجية المثالية.

ولكن دعونا نرَ ما إذا كان يمكننا الرجوع خطوة وتحديد بعض أنماط الأداء العامة...


## أنماط الأداء

تتكون أبسط صفحة من ترميز HTML: بدون CSS أو جافا سكريبت أو أي نوع آخر من الموارد. ولعرض هذه الصفحة، يتعين على المتصفح بدء الطلب، وانتظار وصول مستند HTML، وتحليله، وتصميم DOM، وفي النهاية عرضه على الشاشة:

<pre class="prettyprint">
{% includecode content_path="web/fundamentals/performance/critical-rendering-path/_code/basic_dom_nostyle.html" region_tag="full" adjust_indentation="auto" %}
</pre>

<img src="images/analysis-dom.png" alt="Hello world CRP" class="center">

**يحدد الزمن بين T<sub>0</sub> و T<sub>1</sub> captures the مرات معالجة الشبكة والخادم.** وفي أفضل الحالات (إذا كان ملف HTML صغيرًا)، لن نحتاج إلا إلى انتقال جولة شبكة واحد لجلب المستند بالكامل - نظرًا لكيفية عمل بروتوكولات نقل TCP، قد تتطلب الملفات الكبيرة مزيدًا من الجولات، وسنتناول هذا الموضوع في درس لاحق. **نتيجة لذلك، يمكننا القول بأن الصفحة الواردة أعلاه تتضمن مسار عرض حرج بجولة واحدة (حد أدنى).**

والآن، يمكننا التفكير في الصفحة نفسها ولكن بملف CSS خارجي:

<pre class="prettyprint">
{% includecode content_path="web/fundamentals/performance/critical-rendering-path/_code/analysis_with_css.html" region_tag="full" adjust_indentation="auto" %}
</pre>

<img src="images/analysis-dom-css.png" alt="DOM + CSSOM CRP" class="center">

مجددًا سنتحمل انتقال جولة شبكة لجلب مستند HTML ولنعرف بعد ذلك من الترميز الذي نسترده أننا سنحتاج كذلك إلى ملف CSS: وهذا يعني أنه يتعين على المتصفح الرجوع إلى الخادم والحصول على CSS حتى يتمكن من عرض الصفحة على الشاشة. **نتيجة لذلك، ستتحمل هذه الصفحة انتقال جولتين كحد أدنى حتى يتم عرض الصفحة** - ومجددًا، قد يستغرق ملف CSS عدة جولات، ومن ثم نركز على أنه `كحد أدنى`.

دعونا نعرِّف المفردات التي سنستخدمها لوصف مسار العرض الحرج:

* **المورد الحرج:** هو المورد الذي قد يحظر العرض الأولي للصفحة.
* **طول المسار الحرج:** عدد الجولات أو إجمالي الوقت المطلوب لجلب جميع الموارد الحرجة.
* **وحدات بايت الحرجة:** إجمالي عدد وحدات بايت المطلوبة للحصول على العرض الأول للصفحة، وهو مجموع أحجام ملفات النقل في الموارد الحرجة.
كان المثال الأول المتعلق بصفحة HTML المفردة يتضمن موردًا حرجًا واحدًا (مستند HTML)، كما كان طول المسار الحرج أيضًا يساوي جولة شبكة واحدة (بافتراض أن حجم الملف صغير)، وكان إجمالي عدد وحدات بايت الحرج هو حجم نقل مستند HTML نفسه.

الآن، يمكننا مقارنة ذلك بخصائص المسار الحرج في المثال HTML + CSS الوارد أعلاه:

<img src="images/analysis-dom-css.png" alt="DOM + CSSOM CRP" class="center">

* **2** من الموارد الحرجة
* **2** أو أكثر من الجولات لطول المسار الحرج كحد أدنى
* **9** كيلوبايت من وحدات بايت الحرجة

نحتاج إلى أن ينشئ كل من HTML وCSS شجرة العرض، ونتيجة لذلك يعد كل من HTML وCSS موردًا حرجًا: يتم جلب CSS فقط بعد وصول المتصفح إلى مستند HTML، ومن ثم يكون طول المسار الحرج بجولتين كحد أدنى؛ حيث يضيف الموردان ما يصل إلى إجمالي 9 كيلوبايت من وحدات بايت الحرجة.

والآن يمكننا إضافة ملف جافا سكريبت إضافي في المزيج.

<pre class="prettyprint">
{% includecode content_path="web/fundamentals/performance/critical-rendering-path/_code/analysis_with_css_js.html" region_tag="full" adjust_indentation="auto" %}
</pre>

لقد أضفنا app.js، الذي يعد أصلاً خارجيًا من أصول جافا سكريبت على الصفحة، وكما نعرف من الآن، فهو مورد يحظر المحلل (أي حرج). والأسوأ من ذلك، حتى يمكننا تنفيذ ملف جافا سكريبت، يجب أيضًا أن نحظر CSSOM وننتظره - تذكر أنه يمكن لجافا سكريبت الاستعلام عن CSSOM ومن ثم سيتوقف المتصفح مؤقتًا حتى يتم تنزيل `style.css` ويتم إنشاء CSSOM.

<img src="images/analysis-dom-css-js.png" alt="DOM وCSSOM وJavaScript CRP" class="center">

إلا أنه بشكل عملي عند النظر إلى `تدفق الشبكة` في هذه الصفحة، ستلاحظ أن كلاً من طلبي CSS وجافا سكريبت سيعملان في وقت واحد: يحصل المتصفح على HTML ويكتشف الموردين ويبدأ الطلبين. ونتيجة لذلك، تتضمن الصفحة الواردة أعلاه خصائص المسار الحرج التالية:

* **3** موارد حرجة
* **2** أو أكثر من الجولات لطول المسار الحرج كحد أدنى
* **11** كيلوبايت من وحدات بايت الحرجة

الآن أصبحت لدينا ثلاثة موارد حرجة تضيف حتى 11 كيلوبايت من وحدات بايت الحرجة، إلا أن طول المسار الحرج لا يزال جولتين نظرًا لأنه يمكننا نقل CSS وجافا سكريبت بالتوازي. **يعني التوصل إلى ميزات مسار العرض الحرج التمكن من تحديد الموارد الحرجة وكذلك استيعاب كيفية جدولة المتصفح لعمليات الجلب.** والآن يمكننا استكمال المثال…

بعد طرح الأمر على أحد مطوِّري الموقع اكتشفنا أن جافا سكريبت الذي ضمَّناه في الصفحة لا يحتاج إلى الحظر: ذلك أن لدينا بعض الإحصاءات وشفرة أخرى لا تحتاج إلى حظر عرض الصفحة. وبمعرفة ذلك، أصبح بإمكاننا إضافة السمة `async` إلى علامة النص البرمجي لإلغاء حظر المحلل:

<pre class="prettyprint">
{% includecode content_path="web/fundamentals/performance/critical-rendering-path/_code/analysis_with_css_js_async.html" region_tag="full" adjust_indentation="auto" %}
</pre>

<img src="images/analysis-dom-css-js-async.png" alt="DOM وCSSOM وasync JavaScript CRP" class="center">

هناك العديد من الميزات لجعل النص البرمجي غير متزامن:

* لم يعد النص البرمجي يحظر المحلل وليس جزءًا من مسار العرض الحرج
* نظرًا لأنه ليست هناك أية نصوص برمجية حرجة، فلن يحتاج CSS أيضًا إلى حظر الحدث domContentLoaded
* كلما كان بدء حدث domContentLoaded أسرع، كان بدء تنفيذ منطق التطبيق أسرع

نتيجة لذلك، ترجع الصفحة المحسنة إلى موردين حرجين (HTML وCSS)، بطول مسار حرج يبلغ جولتين كحد أدنى، وإجمالي 9 كيلوبايت من وحدات بايت الحرجة.

وفي الختام، هل يمكن القول بأن ورقة أنماط CSS لم تكن لازمة إلا للطباعة فقط؟ كيف كان سيبدو ذلك؟

<pre class="prettyprint">
{% includecode content_path="web/fundamentals/performance/critical-rendering-path/_code/analysis_with_css_nb_js_async.html" region_tag="full" adjust_indentation="auto" %}
</pre>

<img src="images/analysis-dom-css-nb-js-async.png" alt="DOM, non-blocking CSS, and async JavaScript CRP" class="center">

نظرًا لأن المورد style.css لا يُستخدم إلا للطباعة، فلن يحتاج المتصفح إلى حظره لعرض الصفحة. ومن ثم، بمجرد اكتمال إنشاء DOM، تصبح لدى المتصفح معلومات كافية لعرض الصفحة. ونتيجة لذلك، تتضمن هذه الصفحة موردًا حرجًا واحدًا فقط (مستند HTML)، ويكون طول مسار العرض الحرج جولة واحدة كحد أدنى.