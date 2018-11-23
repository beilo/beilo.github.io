---
title: 从源码学习UEToll–捕获控件梳理+相对位置
date: 2018-11-23 14:38:34
tags: Android
toc: true
---

# 从源码学习UEToll

- [x] 捕获控件梳理
- [x] 相对位置功能梳理
- [x] 网格栅栏功能梳理

## 捕获代码分析
### TransparentActivity
``` java
public @interface Type {
    int TYPE_UNKNOWN = -1;
    int TYPE_EDIT_ATTR = 1; // 捕获控件
    int TYPE_SHOW_GRIDDING = 2; // 网格
    int TYPE_RELATIVE_POSITION = 3; // 相对位置
}

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    /// 省略代码
    type = getIntent().getIntExtra(EXTRA_TYPE, TYPE_UNKNOWN);

    switch (type) {
        case TYPE_EDIT_ATTR: // 捕获控件的入口
            EditAttrLayout editAttrLayout = new EditAttrLayout(this);
            editAttrLayout.setOnDragListener(new EditAttrLayout.OnDragListener() {
                @Override
                public void showOffset(String offsetContent) {
                    board.updateInfo(offsetContent); //更新左下角展示的view描述
                }
            });
            vContainer.addView(editAttrLayout);
            break;
    }
}
```
### EditAttrLayout
``` java
public class EditAttrLayout extends CollectViewsLayout {
    private Element targetElement;
    private AttrsDialog dialog;
    private IMode mode = new ShowMode();
    private float lastX, lastY;
    private OnDragListener onDragListener;   
    
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if (targetElement != null) {
            canvas.drawRect(targetElement.getRect(), areaPaint);
            mode.onDraw(canvas); // 进行mode的draw
        }
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                lastX = event.getX();
                lastY = event.getY();
                break;
            case MotionEvent.ACTION_UP:
                mode.triggerActionUp(event); // 调用ShowMode.triggerActionUp(默认),可更改为MoveMode.triggerActionUp
                break;
            case MotionEvent.ACTION_MOVE:
                mode.triggerActionMove(event); // 移动view时触发的方法
                break;
        }
        return true;
    }


    class ShowMode implements IMode {

        @Override
        public void onDraw(Canvas canvas) {
            Rect rect = targetElement.getRect();
            drawLineWithText(canvas, rect.left, rect.top - lineBorderDistance, rect.right, rect.top - lineBorderDistance);
            drawLineWithText(canvas, rect.right + lineBorderDistance, rect.top, rect.right + lineBorderDistance, rect.bottom);
        }

        @Override
        public void triggerActionMove(MotionEvent event) {

        }

        @Override
        public void triggerActionUp(final MotionEvent event) {
            // 在按下抬起的时候 获取到event,并生成element
            final Element element = getTargetElement(event.getX(), event.getY());
            if (element != null) {
                targetElement = element; // 赋值给当前元素
                invalidate();  // 请求重绘View树,即draw()过程
                if (dialog == null) {
                    dialog = new AttrsDialog(getContext());
                    dialog.setAttrDialogCallback(new AttrsDialog.AttrDialogCallback() {
                        // 里面的回调我们先不关心,现在只关心bottom弹出部分
                    });
                }
                dialog.show(targetElement); // 弹出展示页面
            }
        }
    }
}
```

### AttrsDialog
``` java
public class AttrsDialog extends Dialog {
    private RecyclerView vList; // bottom view
    private Adapter adapter = new Adapter();

    public void show(Element element) {
        show(); // Dialog.show()
        // ... 省略些window配置
        adapter.notifyDataSetChanged(element); // 更新bottom view 
        layoutManager.scrollToPosition(0);
    }    
}
```
到这里发现只是更新了`adapter`数据,那接下来就应该去`adapter`里面去找数据的来源了.
`adapter`代码太多了了截取了相关代码进行展示

### AttrsDialog.Adapter
``` java
public static class Adapter extends RecyclerView.Adapter {

    private List<Item> items = new ItemArrayList<>();
    private AttrDialogCallback callback;

    // 更新item数据
    public void notifyDataSetChanged(Element element) {
        items.clear();
        for (String attrsProvider : UETool.getInstance().getAttrsProvider()) {
            try {
                IAttrs attrs = (IAttrs) Class.forName(attrsProvider).newInstance();
                items.addAll(attrs.getAttrs(element)); // 这里获取的数据
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        notifyDataSetChanged();
    }
}
```
可以发现数据都是通过`IAttrs.getAttrs(element)`获取的.接下来就`Ctrl+Alt+鼠标左键`找到了`UETCore`/`UETTextView`/`UETImageView`,
它们都实现了`IAttrs`类.由下面的代码就可以知道`item`的数据是从哪里来的了.

1. 先是获取到element的view
2. 通过`AttrsManager.createAttrs(view)`获取自定义view的属性,并且进行填充
3. 针对不同的属性填充不同的item,对应的值从view中获取
4. 最后返回一个总属性的`List<item>`

### UETCore
``` java
public class UETCore implements IAttrs {

    @Override
    public List<Item> getAttrs(Element element) {
        List<Item> items = new ArrayList<>();

        View view = element.getView();

        items.add(new SwitchItem("Move", element, SwitchItem.Type.TYPE_MOVE));
        items.add(new SwitchItem("ValidViews", element, SwitchItem.Type.TYPE_SHOW_VALID_VIEWS));

        IAttrs iAttrs = AttrsManager.createAttrs(view);
        if (iAttrs != null) {
            items.addAll(iAttrs.getAttrs(element));
        }

        items.add(new TitleItem("COMMON"));
        items.add(new TextItem("Class", view.getClass().getName()));
        items.add(new TextItem("Id", Util.getResId(view)));
        items.add(new TextItem("ResName", Util.getResourceName(view.getId())));
        items.add(new TextItem("Clickable", Boolean.toString(view.isClickable()).toUpperCase()));
        items.add(new TextItem("Focused", Boolean.toString(view.isFocused()).toUpperCase()));
        items.add(new AddMinusEditItem("Width（dp）", element, EditTextItem.Type.TYPE_WIDTH, px2dip(view.getWidth())));
        items.add(new AddMinusEditItem("Height（dp）", element, EditTextItem.Type.TYPE_HEIGHT, px2dip(view.getHeight())));
        items.add(new TextItem("Alpha", String.valueOf(view.getAlpha())));
        Object background = Util.getBackground(view);
        if (background instanceof String) {
            items.add(new TextItem("Background", (String) background));
        } else if (background instanceof Bitmap) {
            items.add(new BitmapItem("Background", (Bitmap) background));
        }
        items.add(new AddMinusEditItem("PaddingLeft（dp）", element, EditTextItem.Type.TYPE_PADDING_LEFT, px2dip(view.getPaddingLeft())));
        items.add(new AddMinusEditItem("PaddingRight（dp）", element, EditTextItem.Type.TYPE_PADDING_RIGHT, px2dip(view.getPaddingRight())));
        items.add(new AddMinusEditItem("PaddingTop（dp）", element, EditTextItem.Type.TYPE_PADDING_TOP, px2dip(view.getPaddingTop())));
        items.add(new AddMinusEditItem("PaddingBottom（dp）", element, EditTextItem.Type.TYPE_PADDING_BOTTOM, px2dip(view.getPaddingBottom())));

        return items;
    }

    static class AttrsManager {

        public static IAttrs createAttrs(View view) {
            if (view instanceof TextView) {
                return new UETTextView();
            } else if (view instanceof ImageView) {
                return new UETImageView();
            }
            return null;
        }
    }

    static class UETTextView implements IAttrs {

        @Override
        public List<Item> getAttrs(Element element) {
            List<Item> items = new ArrayList<>();
            TextView textView = ((TextView) element.getView());
            items.add(new TitleItem("TextView"));
            items.add(new EditTextItem("Text", element, EditTextItem.Type.TYPE_TEXT, textView.getText().toString()));
            items.add(new AddMinusEditItem("TextSize（sp）", element, EditTextItem.Type.TYPE_TEXT_SIZE, px2sp(textView.getTextSize())));
            items.add(new EditTextItem("TextColor", element, EditTextItem.Type.TYPE_TEXT_COLOR, Util.intToHexColor(textView.getCurrentTextColor())));
            List<Pair<String, Bitmap>> pairs = Util.getTextViewBitmap(textView);
            for (Pair<String, Bitmap> pair : pairs) {
                items.add(new BitmapItem(pair.first, pair.second));
            }
            items.add(new SwitchItem("IsBold", element, SwitchItem.Type.TYPE_IS_BOLD, textView.getTypeface() != null ? textView.getTypeface().isBold() : false));
            return items;
        }
    }

    static class UETImageView implements IAttrs {

        @Override
        public List<Item> getAttrs(Element element) {
            List<Item> items = new ArrayList<>();
            ImageView imageView = ((ImageView) element.getView());
            items.add(new TitleItem("ImageView"));
            items.add(new BitmapItem("Bitmap", Util.getImageViewBitmap(imageView)));
            items.add(new TextItem("ScaleType", Util.getImageViewScaleType(imageView)));
            return items;
        }
    }
}
```
那么`element`从哪里来的呢
``` java
    // AttrsDialog
    public void show(Element element) {
        show();
        Window dialogWindow = getWindow();
        WindowManager.LayoutParams lp = dialogWindow.getAttributes();
        dialogWindow.setGravity(Gravity.LEFT | Gravity.TOP);
        lp.x = element.getRect().left;
        lp.y = element.getRect().bottom;
        lp.width = getScreenWidth() - dip2px(30);
        lp.height = getScreenHeight() / 2;
        dialogWindow.setAttributes(lp);
        adapter.notifyDataSetChanged(element); //这里传递给adapter的
        layoutManager.scrollToPosition(0);
    }

    // EditAttrLayout.ShowMode 
    public void triggerActionUp(final MotionEvent event) {
        final Element element = getTargetElement(event.getX(), event.getY());
        if (element != null) {
            targetElement = element;
            invalidate();
            if (dialog == null) {
                dialog = new AttrsDialog(getContext());
                dialog.setAttrDialogCallback(new AttrsDialog.AttrDialogCallback() {
                    @Override
                    public void enableMove() {
                        mode = new MoveMode();
                        dialog.dismiss();
                    }
                    @Override
                    public void showValidViews(int position, boolean isChecked) {
                        int positionStart = position + 1;
                        if (isChecked) {
                            dialog.notifyValidViewItemInserted(positionStart, getTargetElements(lastX, lastY), targetElement);
                        } else {
                            dialog.notifyItemRangeRemoved(positionStart);
                        }
                    }
                    @Override
                    public void selectView(Element element) {
                        targetElement = element;
                        dialog.dismiss();
                        dialog.show(targetElement);
                    }
                });
                dialog.setOnDismissListener(new DialogInterface.OnDismissListener() {
                    @Override
                    public void onDismiss(DialogInterface dialog) {
                        if (targetElement != null) {
                            targetElement.reset();
                            invalidate();
                        }
                    }
                });
            }
            dialog.show(targetElement);
        }
    }
```
就会发现重点在`getTargetElement`方法里面,所有更新`adapter`的操作都走了这个方法.
跟踪上去
``` java
    protected Element getTargetElement(float x, float y) {
        Element target = null;
        for (int i = elements.size() - 1; i >= 0; i--) {
            final Element element = elements.get(i);
            if (element.getRect().contains((int) x, (int) y)) {
                if (isParentNotVisible(element.getParentElement())) {
                    continue;
                }
                if (element != childElement) {
                    childElement = element;
                    parentElement = element;
                } else if (parentElement != null) {
                    parentElement = parentElement.getParentElement();
                }
                target = parentElement;
                break;
            }
        }
        if (target == null) {
            Toast.makeText(getContext(), getResources().getString(R.string.uet_target_element_not_found, x, y), Toast.LENGTH_SHORT).show();
        }
        return target;
    }
```
发现跟`elements`这个数组有关系
``` java 
    // CollectViewsLayout
    private void traverse(View view) {
        if (UETool.getInstance().getFilterClasses().contains(view.getClass().getName())) return;
        if (view.getAlpha() == 0 || view.getVisibility() != View.VISIBLE) return;
        if (getResources().getString(R.string.uet_disable).equals(view.getTag())) return;
        elements.add(new Element(view));
        if (view instanceof ViewGroup) {
            ViewGroup parent = (ViewGroup) view;
            for (int i = 0; i < parent.getChildCount(); i++) {
                traverse(parent.getChildAt(i));
            }
        }
    }

    
    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        try {
            Activity targetActivity = UETool.getInstance().getTargetActivity();
            WindowManager windowManager = targetActivity.getWindowManager();
            Field mGlobalField = Class.forName("android.view.WindowManagerImpl").getDeclaredField("mGlobal");
            mGlobalField.setAccessible(true);

            if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.M) {
                Field mViewsField = Class.forName("android.view.WindowManagerGlobal").getDeclaredField("mViews");
                mViewsField.setAccessible(true);
                List<View> views = (List<View>) mViewsField.get(mGlobalField.get(windowManager));
                for (int i = views.size() - 1; i >= 0; i--) {
                    View targetView = getTargetDecorView(targetActivity, views.get(i));
                    if (targetView != null) {
                        traverse(targetView);
                        break;
                    }
                }
            } else {
                Field mRootsField = Class.forName("android.view.WindowManagerGlobal").getDeclaredField("mRoots");
                mRootsField.setAccessible(true);
                List viewRootImpls;
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
                    viewRootImpls = (List) mRootsField.get(mGlobalField.get(windowManager));
                } else {
                    viewRootImpls = Arrays.asList((Object[]) mRootsField.get(mGlobalField.get(windowManager)));
                }
                for (int i = viewRootImpls.size() - 1; i >= 0; i--) {
                    Class clazz = Class.forName("android.view.ViewRootImpl");
                    Object object = viewRootImpls.get(i);
                    Field mWindowAttributesField = clazz.getDeclaredField("mWindowAttributes");
                    mWindowAttributesField.setAccessible(true);
                    Field mViewField = clazz.getDeclaredField("mView");
                    mViewField.setAccessible(true);
                    View decorView = (View) mViewField.get(object);
                    WindowManager.LayoutParams layoutParams = (WindowManager.LayoutParams) mWindowAttributesField.get(object);
                    if (layoutParams.getTitle().toString().contains(targetActivity.getClass().getName())
                            || getTargetDecorView(targetActivity, decorView) != null) {
                        traverse(decorView);
                        break;
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

```
发现这个方法通过`反射`获取到应用的`mViews`,然后循环找到当前`Activity`的`decorView`,
然后递归`traverse`方法获取到所有的子`view`,填充到`elements`中,最后被`getTargetElement`获取出来.
然后基本上就这样了.


## 总结-流程
1. `UETMenu.show()`在`window`层面展示一个悬浮框
2. 点击第一个按钮`捕获控件`,跳转到`TransparentActivity`
3. 根据`type`在`Activity`的`vContainer`添加`EditAttrLayout`
4. 在`EditAttrLayout`的`MotionEvent.ACTION_UP`事件触发的时候就执行`ShowModel`的`triggerActionUp`,弹出`AttrsDialog`,用于展示相关`View`的信息
5. `View`的详细信息在`UETCore`中获取
6. 这时候注意到`adapter`的`items`的数据是由`show(element)`传递进来的
7. `show`的`element`是`getTargetElement()`方法提供的
8. `getTargetElement`中可以看到有个`elements`,再找到他的源头`->traverse->onAttachedToWindow`

捕获代码并且展示的这一部分就差不多了.大致流程应该梳理的还算清楚了.


## 相对位置分析
### TransparentActivity
``` java
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ... 
        switch (type) {
            case TYPE_EDIT_ATTR:
                ...
            case TYPE_RELATIVE_POSITION:
                // 这个是入口,将一个RelativePositionLayout,添加到了
                // R.id.container 里面
                vContainer.addView(new RelativePositionLayout(this));
                break;
            case TYPE_SHOW_GRIDDING:
                ...
            default:
                ...
        }
        ...
    }

```

### RelativePositionLayout
``` java
    private final int elementsNum = 2; // 指定View个数为2个
    // View
    private Element[] relativeElements = new Element[elementsNum];
    private int searchCount = 0; // 搜索次数,也就是你点击View的次数
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                break;
            case MotionEvent.ACTION_UP:
                // 在这里处理View的逻辑,被整除的在第一个,否则在第二个,同时
                // searchCount累加
                final Element element = getTargetElement(event.getX(), event.getY());
                if (element != null) {
                    relativeElements[searchCount % elementsNum] = element;
                    searchCount++;
                    invalidate();
                }
                break;
        }
        return true;
    }
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        if (relativeElements == null) {
            return;
        }

        boolean doubleNotNull = true;
        for (Element element : relativeElements) {
            // (1)
            if (element != null) {
                Rect rect = element.getRect();
                canvas.drawLine(0, rect.top, screenWidth, rect.top, dashLinePaint);
                canvas.drawLine(0, rect.bottom, screenWidth, rect.bottom, dashLinePaint);
                canvas.drawLine(rect.left, 0, rect.left, screenHeight, dashLinePaint);
                canvas.drawLine(rect.right, 0, rect.right, screenHeight, dashLinePaint);
                canvas.drawRect(rect, areaPaint);
            } else {
                doubleNotNull = false;
            }
        }

        if (doubleNotNull) {
            // (2)
            Rect firstRect = relativeElements[searchCount % elementsNum].getRect();
            Rect secondRect = relativeElements[(searchCount - 1) % elementsNum].getRect();

            if (secondRect.top > firstRect.bottom) {
                int x = secondRect.left + secondRect.width() / 2;
                drawLineWithText(canvas, x, firstRect.bottom, x, secondRect.top);
            }

            if (firstRect.top > secondRect.bottom) {
                int x = secondRect.left + secondRect.width() / 2;
                drawLineWithText(canvas, x, secondRect.bottom, x, firstRect.top);
            }

            if (secondRect.left > firstRect.right) {
                int y = secondRect.top + secondRect.height() / 2;
                drawLineWithText(canvas, secondRect.left, y, firstRect.right, y);
            }

            if (firstRect.left > secondRect.right) {
                int y = secondRect.top + secondRect.height() / 2;
                drawLineWithText(canvas, secondRect.right, y, firstRect.left, y);
            }

            drawNestedAreaLine(canvas, firstRect, secondRect);
            drawNestedAreaLine(canvas, secondRect, firstRect);
        }
    }

```
  *`onDraw`中的逻辑*
  1. 初始化时`doubleNotNull`返回`false`,直接跳过`onDraw`的逻辑.
  2. 当点击第一个`View`时.走标记为`(1)`处代码,为`View`绘画边框线和虚线,同时`doubleNotNull`返回`false`,跳过`onDraw`.
  3. 当有两个`View`时.走标记为`(2)`处代码,计算两个`View`的相对位置然后根据不同情况画线.

## 网格栅栏分析
### TransparentActivity
``` java
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ... 
        switch (type) {
            case TYPE_EDIT_ATTR:
                ...
            case TYPE_RELATIVE_POSITION:
                ...
            case TYPE_SHOW_GRIDDING:
                // 这个是入口,将一个GriddingLayout,添加到了
                // R.id.container 里面
                vContainer.addView(new GriddingLayout(this));
                board.updateInfo("LINE_INTERVAL: " + DimenUtil.px2dip(GriddingLayout.LINE_INTERVAL, true));
            default:
                ...
        }
        ...
    }

```
这里就是简单的添加了一个间隔为`5dp`网格View.