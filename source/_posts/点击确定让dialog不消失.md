title: 点击确定让dialog不消失
date: 2015-12-18 20:33:36
tags: Android

---

网上找到了，用反射机制可以随时设置dialog是否消失：
使用反射：
在你的setPositiveButton中添加：

```
 /**
     * 不隐藏dialog
     *
     * @param dialog
     */
    private void stillShowDialog(DialogInterface dialog) {
        try {
            Field field = dialog.getClass().getSuperclass().getDeclaredField("mShowing");
            field.setAccessible(true);
            field.set(dialog, false);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 取消dialog
     *
     * @param dialog
     */
    private void dismissDialog(DialogInterface dialog) {
        try {
            Field field = dialog.getClass().getSuperclass().getDeclaredField("mShowing");
            field.setAccessible(true);
            field.set(dialog, true);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```