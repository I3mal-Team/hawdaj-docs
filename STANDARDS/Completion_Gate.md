# Completion_Gate — بوابة الإنجاز (إلزامية)

> **سياسة حاكمة.** لا تُعتبر أي مهمة **مكتملة** إلا إذا تحقّقت **كل** بنود القائمة أدناه. أي بند ناقص → الحالة تبقى **❌ INCOMPLETE**.
> يُرجَع لهذا المستند من الخطوة الأخيرة في كل تدفّق `WORKFLOWS/` ومن `CLAUDE.md §2 (الخطوة 11)`. لا يُكرَّر — يُشار إليه.

---

## القائمة الإلزامية (Completion Checklist)
اعرضها في نهاية كل مهمة معلّمة الحالة:

```
### ✅ Completion Gate — <عنوان المهمة>
- [ ] Implementation completed — التنفيذ تمّ
- [ ] Code reviewed — روجِع (ذاتي/أداة، حسب النطاق)
- [ ] Self-review completed — مراجعة ذاتية (معايير/تسمية/حالة/أخطاء/تعريب)
- [ ] Regression analysis completed — تحليل الانحدار (TESTING/REGRESSION_CHECKLIST + قسم الميزة)
- [ ] Documentation synchronized — التوثيق مزامن مع الكود
- [ ] Feature documentation updated — مستند الميزة (FEATURES/NN_*) مُحدَّث/مُلحَق
- [ ] API documentation updated (if required) — API_INDEX لو تغيّر endpoint/عقد
- [ ] Database documentation updated (if required) — DATABASE_INDEX لو تغيّر جدول/migration
- [ ] FEATURE_GRAPH updated (if affected) — لو تغيّرت التبعيات
- [ ] CHANGE_LOG updated — سطر لهذا التغيير (دائمًا)
- [ ] PROJECT_PROGRESS updated — سجل سردي
- [ ] KNOWN_ISSUES updated (if applicable) — حادثة/عطل تشغيلي سُجِّل أو عُلِّم ✅
- [ ] Related Workflows updated (if needed) — لو تغيّر إجراء
- [ ] **hawdaj-docs committed** — تغييرات التوثيق أُودعت (git commit)
- [ ] **hawdaj-docs pushed** — التوثيق مرفوع (git push) — [[Documentation_Sync]]

الحالة: ✅ COMPLETE  ⟵ فقط لو كل ما سبق ✓ — وإلا ❌ INCOMPLETE
```

> **مزامنة التوثيق ([[Documentation_Sync]]):** الكود والتوثيق في ريبوهات منفصلة → التحديث المحلي **لا يكفي**. لا تكتمل المهمة قبل **commit + push** لـ`hawdaj-docs`. لو التغيير يؤثّر على التوثيق ولم يُرفع بعد، أبلغ:
> ⚠️ تم اكتشاف أن هذا التعديل يؤثر على قاعدة المعرفة المشتركة. يجب تحديث الملفات المناسبة داخل مستودع hawdaj-docs. لا تعتبر المهمة مكتملة حتى يتم تحديث ورفع التوثيق.

## قاعدة الحسم
- **بند واحد ناقص = المهمة ❌ INCOMPLETE.** لا تُعلن الإتمام، لا تُنهِ الجلسة، لا تقل "تمّ".
- "if required / if applicable" تعني: قيّم صراحةً — لو غير منطبق اكتب "لا ينطبق" (يُحتسب ✓)، لا تتجاهله بصمت.
- **مزامنة التوثيق ليست اختيارية** — جزء من التنفيذ (سياسة `CLAUDE.md §2`). المهمة لا تكتمل قبلها.

## متى تُطبَّق
- نهاية **كل** تدفّق تنفيذي: BUG_FIX · NEW_FEATURE · REFACTOR · PERFORMANCE · SECURITY.
- CODE_REVIEW: تُطبَّق لو أدّى إلى تطبيق إصلاحات (`--fix`)؛ وإلا مراجعة قراءة فقط.

## الربط
- يسبقه: [[Impact_Analysis]] → خطة → موافقة → تنفيذ → مراجعة ذاتية → regression.
- يُغذّي: `CHANGE_LOG.md`, `PROJECT_PROGRESS.md`, `KNOWN_ISSUES.md`, `API_INDEX.md`, `DATABASE_INDEX.md`, `FEATURE_GRAPH.md`, مستند الميزة.
- يُستدعى في: `CLAUDE.md §2 (11)` + كل `WORKFLOWS/*`.
