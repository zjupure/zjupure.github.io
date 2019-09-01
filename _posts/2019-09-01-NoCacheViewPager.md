---
layout:     post
title:      "ViewPager禁止预加载"
date:       2019-09-01 12:00:00
author:     "zjupure"
header-img: "img/post-bg-01.jpg"
tags:
- viewpager
- preload

---

### 问题场景

我们知道ViewPager有一个预加载的机制，默认会把ViewPager当前位置的左右相邻的页面预先初始化，默认值是1，可以通过`ViewPager#setOffscreenPageLimit(int)`动态修改，这样做得好处是ViewPager左右滑动更加流畅。

在实际应用中，部分场景或者需求要求用户切换到当前TAB时才初始化页面和数据。比如为了优化APP启动速度，进入首页时默认只加载一个TAB的数据，等数据渲染完成之后再初始化其他TAB；或者某些页面属于实时推荐刷新的数据，预加载请求了后端数据而实际没有曝光，会影响推荐的行为和质量，只有等用户真正浏览到该页面时才处理。这时候就需要禁止ViewPager预加载，按需加载。

### 禁止预加载

刚才提到ViewPager提供了`setOffscreenPageLimit(int)`设置缓存的大小，是否可以设置预加载为0，来解决这个问题呢？经过测试发现，根本没有生效，什么鬼，去看ViewPager的源码，小于1的值被强行改回了1。

```
public class ViewPager extends ViewGroup {
     private static final int DEFAULT_OFFSCREEN_PAGES = 1;
     ......
	/**
     * Set the number of pages that should be retained to either side of the
     * current page in the view hierarchy in an idle state. Pages beyond this
     * limit will be recreated from the adapter when needed.
     *
     * <p>This is offered as an optimization. If you know in advance the number
     * of pages you will need to support or have lazy-loading mechanisms in place
     * on your pages, tweaking this setting can have benefits in perceived smoothness
     * of paging animations and interaction. If you have a small number of pages (3-4)
     * that you can keep active all at once, less time will be spent in layout for
     * newly created view subtrees as the user pages back and forth.</p>
     *
     * <p>You should keep this limit low, especially if your pages have complex layouts.
     * This setting defaults to 1.</p>
     *
     * @param limit How many pages will be kept offscreen in an idle state.
     */
    public void setOffscreenPageLimit(int limit) {
        if (limit < DEFAULT_OFFSCREEN_PAGES) {
            Log.w(TAG, "Requested offscreen page limit " + limit + " too small; defaulting to "
                    + DEFAULT_OFFSCREEN_PAGES);
            limit = DEFAULT_OFFSCREEN_PAGES;
        }
        if (limit != mOffscreenPageLimit) {
            mOffscreenPageLimit = limit;
            populate();
        }
    }
    ......
}
```

ViewPager是通过FragmentPagerAdapter/FragmentStatePagerAdapter去初始化Fragment，关联Fragment生命周期的，预加载流程会执行完Fragment的onCreate/onStart/onResume等生命周期。而Fragment中有一个`setUserVisibleHint(boolean)`方法，告诉应用UI是否对用户可见，在滑动Fragment到用户可见时，PageAdapter会调用这个方法通知Fragment，因为我们可以在onResume中判断Fragment是否真正可见，延迟数据的加载，同时重写`setUserVisibleHint(boolean)`方法来加载数据。

但是，该方案的问题在于，Fragment还是被提前初始化了，onCreateView也执行了，渲染了UI，只是数据逻辑部分延迟了，用户左右滑动TAB时，Fragment实例并没有销毁（即没有调用onDestroyView)。有没有一种可能，让ViewPager连Fragment实例都不创建，真正等到用户滑动到当前TAB时才触发呢。

让我们继续回到`setOffscreenPageLimit(int)`方法，虽然ViewPager默认实现拦截了赋值为0的case，我们能否通过继承ViewPager通过override实现来修改行为实现这一目的呢。mOffscreenPageLimit变量是private的，只能通过反射来动态修改了。populate()是包访问权限的，可以把我们自定义的ViewPager放到相同的support包名下绕开访问控制。自定义的NoCacheViewPager

```
public class NoCacheViewPager extends ViewPager {
	// 默认缓存修改为0
	private static final int DEFAULT_OFFSCREEN_PAGES = 0;
	@Override
	public void setOffscreenPageLimit(int limit) {
	    if (limit > 0) {
	        super.setOffscreenPageLimit(limit);
	    } else {
	        limit = DEFAULT_OFFSCREEN_PAGES;
	        try {
	            int oldLimit = getOffscreenPageLimit();
	            if (limit != oldLimit) {
	                // 通过反射修改mOffscreenPageLimit的值
	                setOffscreenPageLimitByReflect(limit);
	                populate();
	            }
	        } catch (Exception e) {
	            super.setOffscreenPageLimit(limit);
	        }
	    }
	}
	    
	private void setOffscreenPageLimitByReflect(int limit) throws Exception {
	    Field mOffscreenPageLimitField = ViewPager.class.getDeclaredField("mOffscreenPageLimit");
	    mOffscreenPageLimitField.setAccessible(true);
	    mOffscreenPageLimitField.set(this, limit);
	}
}
```

把项目里的ViewPager改为自定义的，试了下，什么还是没有生效，断点跟踪mOffscreenPageLimit的值确实已经改成0了。上网搜索了下，也有人遇到类似问题，说是高版本v4已经修改了实现，但是没有给出解决方案，只是简单拷贝的低版本的v4的代码在使用（https://blog.csdn.net/qq_21898059/article/details/51453938）。

### 源码分析

低版本v4里的ViewPager在滑动操作中是有一些bug的，我们项目用的是28.0.0版本的support v4库，而且copy源码，后续升级维护必然很麻烦，只能继续拔ViewPager的源码了，看怎么workaround。

ViewPager是通过`addNewItem()`方法调用PageAdapter的`instantiateItem()`初始化Fragment实例，进行FragmentTrasication执行生命周期的，找到这个方法的调用处，就能找到为什么设置了预加载为0还能触发左右TAB的实例初始化。

```
ItemInfo addNewItem(int position, int index) {
    ItemInfo ii = new ItemInfo();
    ii.position = position;
    ii.object = mAdapter.instantiateItem(this, position);
    ii.widthFactor = mAdapter.getPageWidth(position);
    if (index < 0 || index >= mItems.size()) {
        mItems.add(ii);
    } else {
        mItems.add(index, ii);
    }
    return ii;
}
```

这个方法在ViewPager类中只在`populate()`中有3处调用，我们分析下具体代码

```
void populate(int newCurrentItem) {
	......
	if (curItem == null && N > 0) {
		// 第一处, 当前item为空，加载的是当前TAB的实例
	    curItem = addNewItem(mCurItem, curIndex);
	}
	
	// Fill 3x the available width or up to the number of offscreen
	// pages requested to either side, whichever is larger.
	// If we have no current item we have no work to do.
	if (curItem != null) {
	    float extraWidthLeft = 0.f;
	    int itemIndex = curIndex - 1;
	    ItemInfo ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
	    final int clientWidth = getClientWidth();
	    final float leftWidthNeeded = clientWidth <= 0 ? 0 :
	            2.f - curItem.widthFactor + (float) getPaddingLeft() / (float) clientWidth;
	    for (int pos = mCurItem - 1; pos >= 0; pos--) {
	        if (extraWidthLeft >= leftWidthNeeded && pos < startPos) {
	            if (ii == null) {
	                break;
	            }
	            if (pos == ii.position && !ii.scrolling) {
	                mItems.remove(itemIndex);
	                mAdapter.destroyItem(this, pos, ii.object);
	                if (DEBUG) {
	                    Log.i(TAG, "populate() - destroyItem() with pos: " + pos
	                            + " view: " + ((View) ii.object));
	                }
	                itemIndex--;
	                curIndex--;
	                ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
	            }
	        } else if (ii != null && pos == ii.position) {
	            extraWidthLeft += ii.widthFactor;
	            itemIndex--;
	            ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
	        } else {
	        	  //这里是第二处, 从for循环的条件看，加载的是当前TAB左侧的实例
	            ii = addNewItem(pos, itemIndex + 1);
	            extraWidthLeft += ii.widthFactor;
	            curIndex++;
	            ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
	        }
	    }
	
	    float extraWidthRight = curItem.widthFactor;
	    itemIndex = curIndex + 1;
	    if (extraWidthRight < 2.f) {
	        ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
	        final float rightWidthNeeded = clientWidth <= 0 ? 0 :
	                (float) getPaddingRight() / (float) clientWidth + 2.f;
	        for (int pos = mCurItem + 1; pos < N; pos++) {
	            if (extraWidthRight >= rightWidthNeeded && pos > endPos) {
	                if (ii == null) {
	                    break;
	                }
	                if (pos == ii.position && !ii.scrolling) {
	                    mItems.remove(itemIndex);
	                    mAdapter.destroyItem(this, pos, ii.object);
	                    if (DEBUG) {
	                        Log.i(TAG, "populate() - destroyItem() with pos: " + pos
	                                + " view: " + ((View) ii.object));
	                    }
	                    ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
	                }
	            } else if (ii != null && pos == ii.position) {
	                extraWidthRight += ii.widthFactor;
	                itemIndex++;
	                ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
	            } else {
	            		// 这里是第3处，从for循环条件看，加载的是当前TAB右侧的实例
	                ii = addNewItem(pos, itemIndex);
	                itemIndex++;
	                extraWidthRight += ii.widthFactor;
	                ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
	            }
	        }
	    }
	
	    calculatePageOffsets(curItem, curIndex, oldCurInfo);
	
	    mAdapter.setPrimaryItem(this, mCurItem, curItem.object);
	}
}
```

看了`populate()`方法，重点在第2处和第3处，分别加载了左侧的TAB和右边的TAB，那么需要修改else的判断条件，让逻辑不触发。仔细分析代码，会看到leftWidthNeeded/rightWidthNeeded判断左右两侧是否有剩余空间可以容纳，如果空间不够，则会触发item的回收操作。在计算这两个值时都用到了2.0f，也就是假定了有两个TAB的场景，滑动时不回收，因此这里的条件也需要根据limit同步修改。

找到根本原因了，接下来就是实现了，由于这部分逻辑都被封装在`populate()`方法中，我们只能拷贝这部分代码通过override方式改写其逻辑了，将populate()代码拷贝到子类之后，会发现有部分private的方法无法访问，没办法了只能继续上反射了。反射是万能的~

改写`populate()`方法之后，只有在pageLimit>0时才添加执行`addNewItem()`操作，否则忽略；默认左右的padding修改为1.0f。

基本功能实现了，发现无法滑动了，又是什么鬼。分析代码发现跟刚才的改动的2.0f有关，滑动时没法添加新的item。再次分析ViewPager的事件拦截机制，当ViewPager检测到左右滑动操作时，会触发populate操作，然后进入SCROLL_STATE_DRAGGING状态。因此我们可以重写`setScrolloState(int)`方法，检测到进入拖拽滑动时，再次触发populate操作，添加新的Item。

populate()方法修改点如下，其他未做修改的地方省略
```
void populate(int newCurrentItem) {
	int offscreenPageLimit = getOffscreenPageLimit();
	if (offscreenPageLimit > 0) {
	    super.populate(newCurrentItem);
	} else {
	    fixPopulate(newCurrentItem);
	}
}

private void fixPopulate(int newCurrentItem) {
	......
	final int pageLimit = getOffscreenPageLimit();
	final boolean isBeingDragged = isBeingDragged();
	// Fill 3x the available width or up to the number of offscreen
	// pages requested to either side, whichever is larger.
	// If we have no current item we have no work to do.
	if (curItem != null) {
	    float extraWidthLeft = 0.f;
	    int itemIndex = curIndex - 1;
	    ItemInfo ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
	    final int clientWidth = getClientWidth();
	    // 这里修改maxWidth为1.0f，下面计算leftWidthNeeded和RightWidthNeeded时会减少
	    final float maxWidth = pageLimit > 0 || isBeingDragged ? 2.f : 1.f;
	    final float leftWidthNeeded = clientWidth <= 0 ? 0 :
	            maxWidth - curItem.widthFactor + (float) getPaddingLeft() / (float) clientWidth;
	    for (int pos = mCurItem - 1; pos >= 0; pos--) {
	        if (extraWidthLeft >= leftWidthNeeded && pos < startPos) {
	            if (ii == null) {
	                break;
	            }
	            if (pos == ii.position && !ii.scrolling) {
	                mItems.remove(itemIndex);
	                mAdapter.destroyItem(this, pos, ii.object);
	                if (DEBUG) {
	                    Log.i(TAG, "populate() - destroyItem() with pos: " + pos
	                            + " instance: " + (ii.object));
	                }
	                itemIndex--;
	                curIndex--;
	                ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
	            }
	        } else if (ii != null && pos == ii.position) {
	            extraWidthLeft += ii.widthFactor;
	            itemIndex--;
	            ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
	        } else if (pageLimit > 0 || isScrollToRight()){
	            // 这里增加条件判断, 只有在pageLimit > 0或者向右滑加载左侧的Item
	            ii = addNewItem(pos, itemIndex + 1);
	            extraWidthLeft += ii.widthFactor;
	            curIndex++;
	            ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
	        }
	    }
	
	    float extraWidthRight = curItem.widthFactor;
	    itemIndex = curIndex + 1;
	    if (extraWidthRight <= maxWidth) {
	        ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
	        final float rightWidthNeeded = clientWidth <= 0 ? 0 :
	                (float) getPaddingRight() / (float) clientWidth + maxWidth;  // maxWidth变成了1.0f
	        for (int pos = mCurItem + 1; pos < N; pos++) {
	            if (extraWidthRight >= rightWidthNeeded && pos > endPos) {
	                if (ii == null) {
	                    break;
	                }
	                if (pos == ii.position && !ii.scrolling) {
	                    mItems.remove(itemIndex);
	                    mAdapter.destroyItem(this, pos, ii.object);
	                    if (DEBUG) {
	                        Log.i(TAG, "populate() - destroyItem() with pos: " + pos
	                                + " instance: " + (ii.object));
	                    }
	                    ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
	                }
	            } else if (ii != null && pos == ii.position) {
	                extraWidthRight += ii.widthFactor;
	                itemIndex++;
	                ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
	            } else if (pageLimit > 0 || isScrollToLeft()){
	                // 这里修改判断条件, 只有在pageLimit > 0或者向左滑加载右侧的Item
	                ii = addNewItem(pos, itemIndex);
	                itemIndex++;
	                extraWidthRight += ii.widthFactor;
	                ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
	            }
	        }
	    }
	
	    calculatePageOffsets(curItem, curIndex, oldCurInfo);
	
	    mAdapter.setPrimaryItem(this, mCurItem, curItem.object);
	}
	.....
}
```

### 最佳实践

到此一个可以禁止预加载的NoCacheViewPager就实现了，通过调用`setOffscreenPageLimit(0)`即可生效，详细的源码见https://github.com/zjupure/WorkSample/blob/master/app/src/main/java/android/support/v4/view/NoCacheViewPager.java

由于反射了ViewPager部分私有变量和方法，release包需要添加如下混淆规则，保证反射逻辑生效。ViewPager是android support的类，反射调用除了些微性能损坏之后，也不会受到Android P的非公开SDK限制，可以放心使用。

```
# Keep ViewPager私有变量, 反射访问
-keep public class android.support.v4.view.ViewPager {
    private int mOffscreenPageLimit;
    private boolean mPopulatePending;
    private ** mItems;
    private void sortChildDrawingOrder();
    private void calculatePageOffsets(...);
}
```







