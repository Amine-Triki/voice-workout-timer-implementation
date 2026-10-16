# Voice Workout Timer

**English** | [العربية](#العربية)

---

## English

A simple Android workout timer with optional voice control. Choose a mode, press **START VOICE SESSION**, get into position, and say **"Go"** when the app shows **Listening for Go**. The timer starts without needing to reach for the phone.

> **Status: early development; working title.** This is a personal project being built and tested in stages, not a finished or broadly validated product. Read [Known limitations](#known-limitations) before relying on it.

### How it works

1. Open the app and choose a mode.
   - **Countdown:** 20, 30, 45 or 60 seconds. An optional tone marks the halfway point, and a different tone signals the end.
   - **Count-up:** measures elapsed time until you press **STOP**. Voice **"Stop"** is available only in builds where it is enabled and the speech engine supports it.
2. Press **START VOICE SESSION** and wait for **Listening for Go**.
3. Get into position and say **"Go"**. The app requests a short confirmation tone and starts the timer; if audio playback fails, the tone may not be heard.
4. In Count-up mode, say **"Stop"** only when the app shows **Listening for Stop**. If voice Stop is unavailable, use the manual **STOP** button. The result remains visible until a new workout starts running, you leave the timer screen, or the app goes to the background.

Manual controls remain available if voice recognition is unavailable. The workout clock does not wait for the speech engine or for a sound to finish playing.

### What makes it different

- **On-device voice processing:** keyword spotting is designed to run on the phone without cloud speech recognition. The app does not send audio to a server.
- **No account or ads:** the app is designed to work offline and has no backend or analytics service in the specified scope.
- **You choose when to start again:** after a set, the default behavior is to return to an idle state. You decide when to start the next set; there is no forced workout plan or fixed rest period.
- **Simple display:** large, high-contrast digits are intended to be readable from a short distance. Accessibility with screen readers and all system font sizes has not been validated.
- **Language-neutral sounds:** start, halfway and finish cues are tones, not spoken audio.

### Who is it for

- **Friends and testers:** thank you for helping. See [Feedback](#feedback).
- **People training at home:** for planks, wall sits, stretching, jump rope and similar exercises in a quiet environment.

### Known limitations

This is an early build. Do not assume voice recognition will behave identically on every device.

- **Android only:** Android 8 / API 26 or newer.
- **Quiet environment:** voice commands are intended for a quiet room without music, television or other people speaking. Noise may cause missed or incorrect detections.
- **English commands only:** the supported command words are **"Go"** and **"Stop"**. Voice Stop depends on the build and engine support; the manual STOP button is the fallback.
- **Foreground use only:** keep the screen on and the app in the foreground during a session. If the app goes to the background—for example, when the phone is locked, Home is pressed, another app is opened, or a call sends the app to the background—the active session is cancelled and the in-progress result is lost.
- **Voice Stop timing is approximate:** a voice-stop result includes the time taken to say the command and the recognition delay. It can therefore be later than a manual stop. Such results are marked with **~**; the amount of delay can vary.
- **Audio feedback is not guaranteed:** a sound may fail to play or may not be audible on a particular device or audio route. The screen state and manual controls remain important fallbacks.
- **Music and Bluetooth audio routes are not supported or validated yet:** for testing, use a quiet room and the phone's normal audio setup.
- **No workout history or settings persistence:** workout results and settings are kept in memory and are not restored after the app closes.
- **Device coverage is limited:** voice accuracy has not been established for every phone, Android version, voice, accent or environment. Only report devices and test outcomes that have actually been tested and recorded.

### Privacy

- Normal workout listening is indicated by **Listening for Go** or, when enabled, **Listening for Stop**. The owner-only diagnostic build may also open the microphone during **Check voice**, with a visible checking/listening indicator.
- When listening is disabled, the app ignores speech events. The underlying microphone capture may take a short time to close; the listening label disappearing does not by itself prove that capture has fully closed.
- Audio is processed on the phone. The app does not save audio recordings or upload audio to a server.
- The app does not require an account and, in the specified scope, does not use analytics or a backend.
- Workout results and settings are not saved. **Exception for the owner's diagnostic build:** it may write a local `diagnostics_log.txt` file containing fault details to help investigate problems. The file is not an audio recording or workout history. Builds intended for friends and public release are configured not to keep this diagnostic log.

### Requirements

- An Android phone running Android 8 or newer, with a microphone.
- Microphone permission for voice control. The app requests it when you start a voice session. If permission is denied or voice is unavailable, the manual controls remain available.

### Try it

- **Friends and testers:** the owner may share a test install file and short instructions. Android may ask you to allow installation from that source.
- **Public release:** planned, but not available yet. A download or store link will be added when a public release is ready.

### Feedback

Please report what happened, including your phone model, Android version, whether **Go** and (if available) **Stop** worked, and the steps that led to a problem. Contact **amine.triki@live.com**. Please do not send audio recordings. Only share diagnostic details after reviewing them for information you do not want to disclose.

### Ideas for later (not promises)

- Multiple rounds in one session with an optional goal, showing completed rounds rather than treating the goal as a requirement.
- Optional saving of Count-up results under an exercise name, stored on the phone.
- Better support for music and Bluetooth audio routes, if it can be implemented and tested reliably.

### For developers

- Built with Flutter. On-device keyword spotting is intended to recognize **"Go"** and, where enabled and supported, **"Stop"** without cloud speech recognition.
- The engineering specification is in [`voice_workout_timer_implementation_v26_patched.md`](voice_workout_timer_implementation_v26_patched.md). Keep this link in sync with the file name committed to the repository.
- **Project license:** no license has been chosen yet. Until a license is added, do not assume that the public repository grants permission to reuse, modify or redistribute the code.
- **Voice-model licensing:** the exact model files and relevant training-data terms are being reviewed. A public release must not include the model until the review is complete and the required distribution status is documented.
- The engineering specification distinguishes owner-only diagnostic builds from builds intended for friends and public release. Do not treat a successful emulator build as proof of voice accuracy on a physical device.

### Disclaimer

This app is a timer, not a coaching or medical tool. Exercise within your own limits and at your own risk.

---

<div dir="rtl" lang="ar">

## العربية

مؤقّت تمارين بسيط لهواتف أندرويد مع إمكانية التحكم الصوتي. اختر الوضع، واضغط **START VOICE SESSION**، وخذ وضعيتك، ثم قل **"Go"** عندما تظهر عبارة **Listening for Go**. يبدأ المؤقّت دون الحاجة إلى لمس الهاتف مرة أخرى.

> **الحالة: مشروع في مرحلة التطوير المبكر وباسم مؤقت.** يُبنى هذا المشروع الشخصي ويُختبر على مراحل، ولم يصبح منتجًا نهائيًا أو مُختبَرًا على نطاق واسع. اقرأ [القيود المعروفة](#القيود-المعروفة) قبل الاعتماد عليه.

### كيف يعمل

1. افتح التطبيق واختر وضعًا.
   - **العدّ التنازلي (Countdown):** 20 أو 30 أو 45 أو 60 ثانية. توجد نغمة اختيارية عند منتصف المدة، ونغمة مختلفة عند النهاية.
   - **العدّ التصاعدي (Count-up):** يقيس الزمن المنقضي حتى تضغط **STOP**. لا يتوفر الإيقاف الصوتي **"Stop"** إلا في النسخ التي تفعّله ويدعمه فيها محرّك التعرف الصوتي.
2. اضغط **START VOICE SESSION** وانتظر ظهور **Listening for Go**.
3. خذ وضعيتك وقل **"Go"**. يطلب التطبيق تشغيل نغمة قصيرة لتأكيد قبول أمر البدء ويبدأ المؤقّت؛ وقد لا تُسمع النغمة إذا تعذّر تشغيل الصوت.
4. في وضع العدّ التصاعدي، قل **"Stop"** فقط عندما تظهر عبارة **Listening for Stop**. إذا لم يكن الإيقاف الصوتي متاحًا، فاستخدم زر **STOP** اليدوي. تبقى النتيجة ظاهرة حتى يبدأ تمرين جديد فعليًا، أو تغادر شاشة المؤقّت، أو ينتقل التطبيق إلى الخلفية.

تظل الأزرار اليدوية متاحة حتى إذا تعذّر التعرف على الصوت. ولا ينتظر مؤقّت التمرين محرّك التعرف الصوتي أو انتهاء تشغيل النغمة كي يتقدم.

### ما الذي يميّزه

- **معالجة صوتية على الهاتف:** صُمّم التعرف على الكلمات المفتاحية ليعمل على الجهاز دون خدمة تعرّف سحابية، ولا يرسل التطبيق الصوت إلى خادم.
- **بلا حساب أو إعلانات:** صُمّم التطبيق ليعمل دون اتصال، ولا يتضمن نطاق المشروع المحدد خادمًا خلفيًا أو خدمة تحليلات.
- **أنت تختار موعد البدء التالي:** بعد انتهاء المجموعة، يعود التطبيق افتراضيًا إلى حالة انتظار. تقرر أنت متى تبدأ المجموعة التالية؛ فلا توجد خطة تدريب مفروضة أو مدة راحة ثابتة.
- **واجهة بسيطة:** صُممت الأرقام الكبيرة وعالية التباين لتكون مقروءة من مسافة قصيرة. لم يُتحقق بعد من إمكانية الاستخدام مع قارئات الشاشة وجميع أحجام الخطوط في النظام.
- **أصوات محايدة لغويًا:** نغمات البداية والمنتصف والنهاية ليست كلمات منطوقة.

### لمن هو

- **الأصدقاء والمختبِرون:** شكرًا لمساعدتكم. راجعوا قسم [الملاحظات](#الملاحظات).
- **من يتمرّن في المنزل:** لتمارين البلانك، والجلوس على الحائط، والتمدد، ونط الحبل، والتمارين المشابهة، في مكان هادئ.

### القيود المعروفة

هذه نسخة مبكرة؛ فلا تفترض أن دقة التعرف الصوتي ستكون متطابقة على جميع الأجهزة.

- **أندرويد فقط:** أندرويد 8 / API 26 أو أحدث.
- **بيئة هادئة:** صُممت الأوامر الصوتية للعمل في غرفة هادئة دون موسيقى أو تلفاز أو أشخاص يتحدثون. قد يتسبب الضجيج في عدم التقاط الأمر أو في التعرف عليه خطأً.
- **أوامر إنجليزية فقط:** كلمتا الأمر هما **"Go"** و**"Stop"**. يعتمد توفر الإيقاف الصوتي على النسخة ودعم المحرّك؛ وزر **STOP** اليدوي هو البديل.
- **العمل في الواجهة الأمامية فقط:** أبقِ الشاشة مضاءة والتطبيق في الواجهة أثناء الجلسة. إذا انتقل التطبيق إلى الخلفية، مثل قفل الهاتف أو الضغط على Home أو فتح تطبيق آخر أو انتقال التطبيق إلى الخلفية بسبب مكالمة، تُلغى الجلسة النشطة ويضيع الزمن غير المكتمل.
- **توقيت الإيقاف الصوتي تقريبي:** تتضمن النتيجة مدة نطق الأمر وتأخّر التعرف عليه، ولذلك قد تكون لاحقة للإيقاف اليدوي. تُميّز هذه النتائج بالرمز **~**، وقد يختلف مقدار التأخير.
- **الموسيقى وتوجيه الصوت عبر Bluetooth غير مدعومين حاليًا:** لم يُتحقق من الأوامر الصوتية مع الموسيقى أو مسارات الصوت عبر Bluetooth. للاختبار، استخدم مكانًا هادئًا وإعداد الصوت المعتاد للهاتف.
- **لا يوجد سجل للتمارين أو حفظ دائم للإعدادات:** تبقى النتائج والإعدادات في الذاكرة ولا تُستعاد بعد إغلاق التطبيق.
- **اختبارات الأجهزة محدودة:** لم تُثبت دقة الأوامر الصوتية على كل هاتف أو إصدار أندرويد أو صوت متحدث أو لهجة أو بيئة. لا تُذكر إلا الأجهزة ونتائج الاختبار التي جرى اختبارها وتسجيلها فعلًا.

### الخصوصية

- يظهر الاستماع العادي للتمرين بعبارة **Listening for Go** أو، عند تفعيل الميزة، **Listening for Stop**. وقد تفتح النسخة التشخيصية الخاصة بالمالك الميكروفون أيضًا أثناء **Check voice** مع إظهار مؤشر واضح للفحص أو الاستماع.
- عند تعطيل الاستماع، يتجاهل التطبيق أحداث الكلام. وقد يستغرق إغلاق التقاط الصوت الفعلي وقتًا قصيرًا؛ لذلك فإن اختفاء عبارة الاستماع لا يثبت وحده أن الالتقاط قد أُغلق بالكامل.
- تتم معالجة الصوت على الهاتف. لا يحفظ التطبيق تسجيلات صوتية ولا يرفع الصوت إلى خادم.
- لا يتطلب التطبيق حسابًا، ولا يتضمن نطاق المشروع المحدد تحليلات أو خادمًا خلفيًا.
- لا تُحفظ نتائج التمارين والإعدادات. **استثناء نسخة المالك التشخيصية:** قد تكتب ملفًا محليًا باسم `diagnostics_log.txt` يتضمن تفاصيل الأعطال للمساعدة في التحقيق فيها. هذا الملف ليس تسجيلًا صوتيًا ولا سجلًا لنتائج التمارين. وتُضبط النسخ المخصصة للأصدقاء والنشر العام بحيث لا تحتفظ بهذا السجل التشخيصي.

### المتطلبات

- هاتف أندرويد يعمل بإصدار أندرويد 8 أو أحدث ويحتوي على ميكروفون.
- إذن الميكروفون للتحكم الصوتي. يطلب التطبيق الإذن عند بدء جلسة صوتية. إذا رُفض الإذن أو تعذّر الصوت، تظل الأزرار اليدوية متاحة.

### كيف تجرّبه

- **الأصدقاء والمختبِرون:** قد يشارك المالك ملف تثبيت تجريبيًا وتعليمات قصيرة. قد يطلب أندرويد السماح بالتثبيت من ذلك المصدر.
- **الإصدار العام:** مخطط له لكنه غير متاح بعد. سيُضاف رابط التنزيل أو المتجر عند تجهيز إصدار عام.

### الملاحظات

أخبرني بما حدث، مع ذكر طراز الهاتف وإصدار أندرويد، وهل عمل الأمر **Go** وأمر **Stop** إن كان متاحًا، والخطوات التي أدت إلى المشكلة. يمكنك التواصل عبر **amine.triki@live.com**. يُرجى عدم إرسال تسجيلات صوتية. ولا تشارك التفاصيل التشخيصية قبل مراجعتها للتأكد من أنها لا تتضمن معلومات لا تريد الإفصاح عنها.

### أفكار للمستقبل (ليست وعودًا)

- عدة جولات في جلسة واحدة مع هدف اختياري، مع عرض عدد الجولات المنجزة فعلًا دون اعتبار الهدف إلزامًا.
- حفظ اختياري لنتائج العدّ التصاعدي تحت اسم تمرين، على الهاتف نفسه.
- تحسين دعم الموسيقى ومسارات الصوت عبر Bluetooth، إذا أمكن تنفيذ ذلك واختباره بصورة موثوقة.

### للمطوّرين

- التطبيق مبني باستخدام Flutter. يهدف التعرف على الكلمات المفتاحية إلى العمل على الجهاز لكلمتي **"Go"** و**"Stop"** عند تفعيل الإيقاف الصوتي ودعمه، دون خدمة تعرّف سحابية.
- توجد المواصفة الهندسية في [`voice_workout_timer_implementation_v26_patched.md`](voice_workout_timer_implementation_v26_patched.md). احرص على أن يطابق هذا الرابط اسم الملف المرفوع إلى المستودع.
- **ترخيص المشروع:** لم يُحدد بعد. إلى حين إضافة ملف ترخيص، لا تفترض أن وجود المستودع علنًا يمنح إذنًا بإعادة استخدام الكود أو تعديله أو إعادة توزيعه.
- **ترخيص النموذج الصوتي:** لا تزال ملفات النموذج المحددة وشروط بيانات التدريب ذات الصلة قيد المراجعة. يجب عدم تضمين النموذج في إصدار عام قبل اكتمال المراجعة وتوثيق أن وضع التوزيع مستوفٍ للشروط.
- تميّز المواصفة الهندسية بين النسخة التشخيصية الخاصة بالمالك والنسخ المخصصة للأصدقاء والنشر العام. نجاح البناء على المحاكي لا يثبت دقة التعرف الصوتي على جهاز حقيقي.

### تنبيه

هذا التطبيق مؤقّت، وليس أداة تدريب أو أداة طبية. تمرّن ضمن حدودك وعلى مسؤوليتك.

</div>
