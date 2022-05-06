> 本文旨在从源码及测试上对UITabview的重用机制进行探讨, 欢迎评论
>
> @[toc]



### 1. Chameleon

&emsp;&emsp;在日常使用UITableView的时候，也许会遇到单元格cell错乱的情况，也许我们也许感叹UITableView的流畅，这其实都来自于它的的复用机制。网上都是从demo中进行实验得出，这里我们从逆向工程中的源码出发，加上实际测试进行重用机制的探索。

&emsp;&emsp;[【Chameleon】](https://github.com/BigZaphod/Chameleon)是一个开源项目，用于将 iOS 的功能迁移到 macOS 上，并且在其中为 macOS 实现了一套与 iOS UIKit 同名的框架，感兴趣的同学可以进行参考，这里也是分析其中UITableView的代码。

#### 1.2 核心代码

```objective-c
// 我们直接省略计算相关的代码吧，只在重用的时候进行分析
- (void)_layoutTableView {
  	/*
  		这里是计算单元格和UITableView的size相关
  		省略
  	*/
  
    // availableCell 记录当前正在显示的 Cell
    // 在滑出显示区之后将添加至 _reusableCells
  	// _cachedCells是当前显示在屏幕上的cell
    NSMutableDictionary *availableCells = [_cachedCells mutableCopy];
    const NSInteger numberOfSections = [_sections count];
    [_cachedCells removeAllObjects];

    // 滑动列表，更新当前显示容器
    for (NSInteger section=0; section<numberOfSections; section++) {
        CGRect sectionRect = [self rectForSection:section];
        tableHeight += sectionRect.size.height;
        if (CGRectIntersectsRect(sectionRect, visibleBounds)) {
           /*省略headerview 和 footerview相关*/
            UITableViewSection *sectionRecord = [_sections objectAtIndex:section];
            const NSInteger numberOfRows = sectionRecord.numberOfRows;

            for (NSInteger row=0; row<numberOfRows; row++) {
                // 构造 indexPath 为代理方法准备
                NSIndexPath *indexPath = [NSIndexPath indexPathForRow:row inSection:section];
                // 获取第 row 个坐标位置
                CGRect rowRect = [self rectForRowAtIndexPath:indexPath];
                // 判断当前 Cell 是否与显示区域相交
              	// 即就是是否显示在屏幕上，一丢丢也算
                if (CGRectIntersectsRect(rowRect,visibleBounds) && rowRect.size.height > 0) {
                    // 首先查看 availableCells 中是否已经有了当前 Cell 的存储
                  	// availableCells是之前_cachedCells copy来的
                  	// 因此是检测是否在屏幕上
                    // 如果没有，则请求 tableView 的代理方法获取 Cell
                    UITableViewCell *cell = [availableCells objectForKey:indexPath] ?: [self.dataSource tableView:self cellForRowAtIndexPath:indexPath];
                    /* 这里需要注意一点：UITableView在首次渲染到屏幕上时，屏幕上的cell，并没有加入到缓存，因此是需要创建的，所有会走代理*/
                    if (cell) {
                        // 获取到 Cell 后，将其进行缓存操作
                        [_cachedCells setObject:cell forKey:indexPath];
                        [availableCells removeObjectForKey:indexPath];
                        cell.highlighted = [_highlightedRow isEqual:indexPath];
                        cell.selected = [_selectedRow isEqual:indexPath];
                        cell.frame = rowRect;
                        cell.backgroundColor = self.backgroundColor;
                        [cell _setSeparatorStyle:_separatorStyle color:_separatorColor];
                        [self addSubview:cell];
                    }
                }
            }
        }
    }

    // 将已经退出屏幕且定义 reuseIdentifier 的 Cell 加入可复用 Cell 容器中
  	// 滑出屏幕的，但是我们进行注册了的cell添加到_reusableCells
    for (UITableViewCell *cell in [availableCells allValues]) {
        if (cell.reuseIdentifier) {
            [_reusableCells addObject:cell];
        } else {
            [cell removeFromSuperview];
        }
    }

    // 不能复用的 Cell 会直接销毁，可复用的 Cell 会存储在 _reusableCells

    // 确保所有的可用（未出现在屏幕上）的复用单元格在 availableCells 中
    // 这样缓存的目的之一是确保动画的流畅性。在动画的帧上都会对显示部分进行处理，重新计算可见 Cell。
    // 如果直接删除掉所有未出现在屏幕上的单元格，在视觉上会观察到突然消失的动作
    // 整体动画具有跳跃性而显得不流畅

    // 把在可视区的 Cell（但不在屏幕上）已经被回收为可复用的 Cell 从视图中移除
    NSArray* allCachedCells = [_cachedCells allValues];
    for (UITableViewCell *cell in _reusableCells) {
        if (CGRectIntersectsRect(cell.frame,visibleBounds) && ![allCachedCells containsObject: cell]) {
            [cell removeFromSuperview];
        }
    }
}
```

#### 1.3 细节

```objective-c
UITableViewCell *cell = [availableCells objectForKey:indexPath] ?: [self.dataSource tableView:self cellForRowAtIndexPath:indexPath];
```

```objective-c
-(UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{

   NSString *identifier=@"dentifier";
    UITableViewCell *cell=[tableView dequeueReusableCellWithIdentifier:identifier];
    if (!cell) {
        cell=[[UITableViewCell alloc]initWithStyle:UITableViewCellStyleDefault reuseIdentifier:identifier];
        // ...
    }
    return cell;
}
```

```objc
- (UITableViewCell *)dequeueReusableCellWithIdentifier:(NSString *)identifier
{
    for (UITableViewCell *cell in _reusableCells) {
        if ([cell.reuseIdentifier isEqualToString:identifier]) {
            UITableViewCell *strongCell = cell;
            
            // the above strongCell reference seems totally unnecessary, but without it ARC apparently
            // ends up releasing the cell when it's removed on this line even though we're referencing it
            // later in this method by way of the cell variable. I do not like this.
            [_reusableCells removeObject:cell];

            [strongCell prepareForReuse];
            return strongCell;
        }
    }
    
    return nil;
}
```



&emsp;&emsp;这里的代码一定要注意，上面的注释已经说明。这里在扣一下细节：

&emsp;&emsp;首先就是我们第一次UITableView渲染到屏幕上的时候，显示在屏幕上的cell并没有加到缓存中，因此都需要创建`(是在显示区域的，不是所有的cell)`。后面在我们滑动的时候，会开始加入到缓存。

&emsp;&emsp;这里也体现了，并不是所有的cell都会进行创建，只是创建屏幕上可见的。

### 2. win

下面是微软实现的一个 `UITableView`，也可以来简单看看。

#### 2.1 核心结构

```c++
struct UITableViewPriv {
    id<UITableViewDelegate> _delegate; // this should be idweak
    id<UITableViewDataSource> _dataSource;
    idretain _footerView;
    idretain _externalObjects;
    float _footerYPos;
    StrongId<UIView> _headerView;
    StrongId<UIView> _backgroundView;
  // 这里就是复用的cell
  // key 是 identifier
  // value 是 cell的数组
    std::unordered_map<std::string, std::vector<ReusableCell*>> _reusableCells; 
    StrongId<NSMutableDictionary> _reusableCellNibs;
    StrongId<NSMutableDictionary> _reusableHeaderClasses;
    StrongId<NSMutableDictionary> _reusableCellClasses;

    float _defaultRowHeight;
    float _defaultSectionHeaderHeight;

    TableViewNode* _rootNode;
    BOOL _needsReload;
    DWORD _isEnumerating;

    VisibleComponents* _visibleComponents;

    UITableViewStyle _style;
    unsigned _separatorStyle;
    BOOL _isEditing;

    DWORD _allowsSelection;
    bool _allowsMultipleSelection, _allowsMultipleSelectionDuringEditing;

    NSInteger sectionCount();
    TableViewSection* sectionAtIndex(int idx);

    CGSize _lastSize;

    void removeReusableCell(ReusableCell* cell);
    void addReusableCell(id cell);
    void addReusableCell(TableViewRow* row);
};
```

```objective-c
@interface UITableView () <UITableViewDelegate> {
@public
    id _dataSource;
    id<UICollectionViewDelegate> _collectionViewDelegate;
    idretain _supplementaryViewReuseQueues, _cellReuseQueues, _decorationViewReuseQueues;
    idretain _nibLayout, _nibCellsExternalObjects, _supplementaryViewsExternalObjects;
    idretain _cellNibDict, _supplementaryViewNibDict, _supplementaryViewClassDict, _cellClassDict;
    UICollectionViewData* _collectionViewData;
    idretain _layout, _allVisibleViewsDict;
    idretain _backgroundView;
    idretain _updateCompletionHandler;

    idretain _insertItems;
    idretain _deleteItems;
    idretain _reloadItems;
    idretain _moveItems;
    idretain _update;
  // 内部实际操作的就是这个 UITableViewPriv
    struct UITableViewPriv* tablePriv;
    idretain _indexPathsForSelectedItems, _indexPathsForHighlightedItems;
    idretain _currentIndexPath, _touchingIndexPath;

    idretain _originalInsertItems, _originalDeleteItems;

    int _reloadingSuspendedCount, _updateCount;

    struct {
        unsigned int delegateShouldHighlightItemAtIndexPath : 1;
        unsigned int delegateDidHighlightItemAtIndexPath : 1;
        unsigned int delegateDidUnhighlightItemAtIndexPath : 1;
        unsigned int delegateShouldSelectItemAtIndexPath : 1;
        unsigned int delegateShouldDeselectItemAtIndexPath : 1;
        unsigned int delegateDidSelectItemAtIndexPath : 1;
        unsigned int delegateDidDeselectItemAtIndexPath : 1;
        unsigned int delegateSupportsMenus : 1;
        unsigned int delegateDidEndDisplayingCell : 1;
        unsigned int delegateDidEndDisplayingSupplementaryView : 1;
        unsigned int dataSourceNumberOfSections : 1;
        unsigned int dataSourceViewForSupplementaryElement : 1;
        unsigned int reloadSkippedDuringSuspension : 1;
        unsigned int scheduledUpdateVisibleCells : 1;
        unsigned int scheduledUpdateVisibleCellLayoutAttributes : 1;
        unsigned int allowsSelection : 1;
        unsigned int allowsMultipleSelection : 1;
        unsigned int updating : 1;
        unsigned int fadeCellsForBoundsChange : 1;
        unsigned int updatingLayout : 1;
        unsigned int needsReload : 1;
        unsigned int reloading : 1;
        unsigned int skipLayoutDuringSnapshotting : 1;
        unsigned int layoutInvalidatedSinceLastCellUpdate : 1;
        unsigned int doneFirstLayout : 1;
    } _collectionViewFlags;
}

- (void)_forwardCellRemoval:(UITableViewCell*)cell;
- (void)_cellSelectedUp:(UITableViewCell*)cell;
- (void)_cellSelectedDown:(UITableViewCell*)cell;
- (void)_cellSelectedCancelled:(UITableViewCell*)cell;
- (void)_cellAccessoryTapped:(UITableViewCell*)cell;
- (BOOL)_canEditCell:(UITableViewCell*)cell;

@end
```



#### 2.2 复用核心

```c++
/*
  cell
*/
class ReusableCell {
public:
    ReusableCell() {
        sourceRow = NULL;
    }
    ~ReusableCell() {
        _cell = nil;
        _reuseIdentifier = nil;
    }

    TableViewRow* sourceRow;
    idretain _cell;
    idretain _reuseIdentifier;
};
/*
	添加复用的 cell
*/
void UITableViewPriv::addReusableCell(id cell) {
    NSString* reuse = [cell reuseIdentifier];
    assert(reuse != nil);

    auto& arr = _reusableCells[[reuse UTF8String]];

    if (arr.size() <= 5) {
        ReusableCell* newReusableCell = new ReusableCell();

        newReusableCell->sourceRow = NULL;
        newReusableCell->_cell = cell;
        [newReusableCell->_cell setHidden:TRUE];
        newReusableCell->_reuseIdentifier = reuse;
        arr.push_back(newReusableCell);
    } else {
        [cell removeFromSuperview];
    }
}
/*
	删除复用的 cell
*/
void UITableViewPriv::removeReusableCell(ReusableCell* cell) {
    id reuse = cell->_reuseIdentifier;
    assert(reuse != nil);

    auto it = _reusableCells.find([reuse UTF8String]);
    if (it != _reusableCells.end()) {
        auto& arr = it->second;
        arr.erase(std::remove(arr.begin(), arr.end(), cell), arr.end());
    }

    delete cell;
}

```



### . apple

&emsp;&emsp;这里只是GitHub上一个开源的逆向工程，并不是苹果公司真正实现的。其实如果我们测试的话，其实会发现，如果你的cell的个数大于15的时候，且你的屏幕也不显示15个以上的cell，苹果的UITableView也会直接创建15个cell，可能是由于成本不大或者其他性能的考虑。