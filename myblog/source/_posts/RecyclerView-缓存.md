---
title: RecyclerView 缓存
date: 2018-08-13 14:18:59
tags: Android
---
- [目录](#%E7%9B%AE%E5%BD%95)
    - [初始代码分析 getViewForPosition](#%E5%88%9D%E5%A7%8B%E4%BB%A3%E7%A0%81%E5%88%86%E6%9E%90-getviewforposition)
    - [RecycleView的四级缓存](#recycleview%E7%9A%84%E5%9B%9B%E7%BA%A7%E7%BC%93%E5%AD%98)
        - [一级缓存 getChangedScrapViewForPosition & getScrapOrHiddenOrCachedHolderForPosition & getScrapOrCachedViewForId](#%E4%B8%80%E7%BA%A7%E7%BC%93%E5%AD%98-getchangedscrapviewforposition--getscraporhiddenorcachedholderforposition--getscraporcachedviewforid)
        - [二级缓存 getScrapOrHiddenOrCachedHolderForPosition & getScrapOrCachedViewForId](#%E4%BA%8C%E7%BA%A7%E7%BC%93%E5%AD%98-getscraporhiddenorcachedholderforposition--getscraporcachedviewforid)
        - [三级缓存 mViewCacheExtension.getViewForPositionAndType](#%E4%B8%89%E7%BA%A7%E7%BC%93%E5%AD%98-mviewcacheextensiongetviewforpositionandtype)
        - [四级缓存 getRecycledViewPool().getRecycledView(type)](#%E5%9B%9B%E7%BA%A7%E7%BC%93%E5%AD%98-getrecycledviewpoolgetrecycledviewtype)
# 目录


## 初始代码分析 getViewForPosition
```
View getViewForPosition(int position, boolean dryRun) {
    return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
}

@Nullable
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
                boolean dryRun, long deadlineNs) {
    ViewHolder holder = null;
    
    1. 从mChangedScrap缓存中获取ViewHodler
    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
    }
   
    1. 从mAttachedScrap取出hiddden但是没有removed的view / mCachedViews取出viewHold 
    通过position
    if (holder == null) {
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
    }

    if (holder == null) {
        final int type = mAdapter.getItemViewType(offsetPosition);
        if (mAdapter.hasStableIds()) {
            1. 从mAttachedScrap  mCachedViews 取出viewhodler 
            通过id position的检索成本比id低
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                    type, dryRun);
        }
        if (holder == null && mViewCacheExtension != null) {
            1. mViewCacheExtension.getViewForPositionAndType获取view 取出viewhodler
            final View view = mViewCacheExtension
                    .getViewForPositionAndType(this, position, type);
            if (view != null) {
                4.1 取出对应view的viewHolder
                holder = getChildViewHolder(view);
            }
        }
        if (holder == null) { 
            1. 从RecycledViewPool中取出对应type的viewholder
            holder = getRecycledViewPool().getRecycledView(type);
        }
        if (holder == null) {
            1. mAdapter.createViewHolder
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
        }
    }

    boolean bound = false;
    if (mState.isPreLayout() && holder.isBound()) {
        // do not update unless we absolutely have to.
        holder.mPreLayoutPosition = position;
    } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        1. bindViewHolder
        bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
    }
    return holder;
}
```

由上面的代码可以发现执行的先后顺序如下:
1. 从`mChangedScrap`缓存中获取`ViewHolder`。
2. 通过`position`，从`mAttachedScrap`或者`mCachedViews中`取出`ViewHolder`。
3. 通过`id`，从`mAttachedScrap`或者`mCachedViews中`取出`ViewHolder`。
4. 通过`mViewCacheExtension`获取到`view`，然后取出对应view的ViewHolder。
5. 从`mRecyclerPool`中取出对应`type`的`ViewHodler`。
6. 调用`mAdapter.createViewHolder`。
7. 当`!holder.isBound() || holder.needsUpdate() || holder.isInvalid()`，也就是`holder`不在范围内或者需要更新或者已经无效时，调用`bindViewHolder`，否则不更新。

## RecycleView的四级缓存
1. 由`mChangedScrap`和`mAttachedScrap`组成的一级缓存,管理数据已经变化和没有分离的`ViewHolder`列表. (屏幕内缓存)
2. 由`mCachedViews`组成的二级缓存,大小由`mViewCacheMax`决定,默认为2. (屏幕外缓存)
3. `ViewCacheExtension`实现自定义缓存.(自定义缓存)
4. `ViewHolder`在首先会缓存在`mCachedViews`中，当超过了个数（比如默认为2）， 就会添加到`RecycledViewPool`中。`RecycledViewPool`会根据每个`ViewType`把`ViewHolder`分别存储在不同的列表中，每个`ViewType`最多缓存`DEFAULT_MAX_SCRAP = 5`个`ViewHolder`，如果`RecycledViewPool`没有被多个`RecycledView`共享，对于线性布局，每个`ViewType`最多只有一个缓存，如果是网格有多少行就缓存多少个。(缓存池)

### 一级缓存 getChangedScrapViewForPosition & getScrapOrHiddenOrCachedHolderForPosition & getScrapOrCachedViewForId
```
ViewHolder getChangedScrapViewForPosition(int position) {
    final int changedScrapSize;
    if (mChangedScrap == null || (changedScrapSize = mChangedScrap.size()) == 0) {
        return null;
    }
    通过position找到mChangedScrap中没有添加FLAG_RETURNED_FROM_SCRAP的holder
    for (int i = 0; i < changedScrapSize; i++) {
        final ViewHolder holder = mChangedScrap.get(i);
        if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position) {
            holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
            return holder;
        }
    }
    通过id找到mChangedScrap中没有添加FLAG_RETURNED_FROM_SCRAP的holder
    if (mAdapter.hasStableIds()) {
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        if (offsetPosition > 0 && offsetPosition < mAdapter.getItemCount()) {
            final long id = mAdapter.getItemId(offsetPosition);
            for (int i = 0; i < changedScrapSize; i++) {
                final ViewHolder holder = mChangedScrap.get(i);
                if (!holder.wasReturnedFromScrap() && holder.getItemId() == id) {
                    holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
                    return holder;
                }
            }
        }
    }
    return null;
}

ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
    final int scrapCount = mAttachedScrap.size();
    从 mAttachedScrap中获取到 ViewHolder,通过position
    for (int i = 0; i < scrapCount; i++) {
        final ViewHolder holder = mAttachedScrap.get(i);
        if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position
                && !holder.isInvalid() && (mState.mInPreLayout || !holder.isRemoved())) {
            holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
            return holder;
        }
    }
    if (!dryRun) {
        取出hiddden但是没有removed的view
        View view = mChildHelper.findHiddenNonRemovedView(position);
        if (view != null) {
            final ViewHolder vh = getChildViewHolderInt(view);
            mChildHelper.unhide(view);
            int layoutIndex = mChildHelper.indexOfChild(view);
            if (layoutIndex == RecyclerView.NO_POSITION) {
                throw new IllegalStateException("layout index should not be -1 after "
                        + "unhiding a view:" + vh + exceptionLabel());
            }
            mChildHelper.detachViewFromParent(layoutIndex);
            scrapView(view);
            vh.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP
                    | ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
            return vh;
        }
    }
    return null;
}

ViewHolder getScrapOrCachedViewForId(long id, int type, boolean dryRun) {
    从mAttachedScrap中通过匹配id获取holder缓存。
    final int count = mAttachedScrap.size();
    for (int i = count - 1; i >= 0; i--) {
        final ViewHolder holder = mAttachedScrap.get(i);
        if (holder.getItemId() == id && !holder.wasReturnedFromScrap()) {
            if (type == holder.getItemViewType()) {
                holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
                if (holder.isRemoved()) {
                    if (!mState.isPreLayout()) {
                        holder.setFlags(ViewHolder.FLAG_UPDATE, ViewHolder.FLAG_UPDATE
                                | ViewHolder.FLAG_INVALID | ViewHolder.FLAG_REMOVED);
                    }
                }
                return holder;
            } else if (!dryRun) {
                mAttachedScrap.remove(i);
                removeDetachedView(holder.itemView, false);
                quickRecycleScrapView(holder.itemView);
            }
        }
    }
    return null;
}

```

### 二级缓存 getScrapOrHiddenOrCachedHolderForPosition & getScrapOrCachedViewForId
```
ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
    从mCachedViews取出ViewHodler,通过position
    final int cacheSize = mCachedViews.size();
    for (int i = 0; i < cacheSize; i++) {
        final ViewHolder holder = mCachedViews.get(i);
        holder有效,则移出一级缓存
        if (!holder.isInvalid() && holder.getLayoutPosition() == position) {
            if (!dryRun) {
                mCachedViews.remove(i);
            }
            return holder;
        }
    }
    return null;
}

ViewHolder getScrapOrCachedViewForId(long id, int type, boolean dryRun) {
    从mCachedViews中通过匹配id获取holder缓存。 
    final int cacheSize = mCachedViews.size();
    for (int i = cacheSize - 1; i >= 0; i--) {
        final ViewHolder holder = mCachedViews.get(i);
        if (holder.getItemId() == id) {
            if (type == holder.getItemViewType()) {
                if (!dryRun) {
                    mCachedViews.remove(i);
                }
                return holder;
            } else if (!dryRun) {
                recycleCachedViewAt(i);
                return null;
            }
        }
    }
    return null;
}
```

### 三级缓存 mViewCacheExtension.getViewForPositionAndType
```
public abstract static class ViewCacheExtension {
    public abstract View getViewForPositionAndType(Recycler recycler, int position, int type);
}
```

### 四级缓存 getRecycledViewPool().getRecycledView(type)
```
static class ScrapData {
    ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
    int mMaxScrap = DEFAULT_MAX_SCRAP;
    long mCreateRunningAverageNs = 0;
    long mBindRunningAverageNs = 0;
}
SparseArray<ScrapData> mScrap = new SparseArray<>();

public ViewHolder getRecycledView(int viewType) {
    RecycledViewPool是根据viewType获取ViewHolder,返回列表最后一个
    final ScrapData scrapData = mScrap.get(viewType);
    if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
        final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
        返回列表最后一个
        return scrapHeap.remove(scrapHeap.size() - 1);
    }
    return null;
}
```


