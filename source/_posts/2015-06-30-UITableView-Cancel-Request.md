title: UITableView Cancel Request
categories: IOS
tags:
  - IOS
  - UITableView
description: UITableview 取消请求
date: 2015-06-30 09:22:03
author:
photos:
---

### 仅加载可见Cell的图片
load images for just the visible rows in viewDidLoad and when the user stops scrolling
```
-(void)viewDidLoad{
    for (NSIndexPath *indexPath in [self.tableView indexPathsForVisibleRows]) {
        [self loadImageForCellAtPath:indexPath];
    }
}

-(void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView {
    for (NSIndexPath *indexPath in [self.tableView indexPathsForVisibleRows]) {
        [self loadImageForCellAtPath:indexPath];
    }
}
```

### 取消Request --- NSBlockOperation

```
self.operationQueue = [[NSOperationQueue alloc] init];
[self.operationQueue setMaxConcurrentOperationCount:NSOperationQueueDefaultMaxConcurrentOperationCount];

// ...

-(void)loadImageForCellAtPath:(NSIndexPath *)indexPath {
    __block NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
        if (![operation isCancelled]) {
            NSString *galleryTinyImageUrl = [[self.smapi getImageUrls:imageId imageKey:imageKey] objectForKey:@"TinyURL"];
            NSData *imageData = [[NSData alloc] initWithContentsOfURL:[NSURL URLWithString:galleryTinyImageUrl]];
            dispatch_async(dispatch_get_main_queue(), ^{
                if (imageData != nil) {
                    UITableViewCell *cell = [tableView cellForRowAtIndexPath:indexPath];
                    cell.imageView.image = [UIImage imageWithData:imageData];
                }
            });
        }
    }];

    NSValue *nonRetainedOperation = [NSValue valueWithNonretainedObjectValue:operation];
    [self.operations addObject:nonRetainedOperation forKey:indexPath];
    [self.operationQueue addOperation:operation];
}
```
Here operations is an NSMutableDictionary. When you want to cancel an operation, you retrieve it by the cell's indexPath, cancel it, and remove it from the dictionary:

```
NSValue *operationHolder = [self.operations objectForKey:indexPath];
NSOperation *operation = [operationHolder nonretainedObjectValue];
[operation cancel];
```


### prefetch the images ---  SDWebImagePrefetcher
```
[self.tableView reloadData];
dispatch_async(dispatch_get_main_queue(), ^{
    [self prefetchImagesForTableView:self.tableView];
});
```

anytime I stop scrolling, I do the same:
```
- (void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView {
    [self prefetchImagesForTableView:self.tableView];
}

- (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate {
    if (!decelerate)
        [self prefetchImagesForTableView:self.tableView];
}
```

```
#pragma mark - Prefetch cells

static NSInteger const kPrefetchRowCount = 10;

/** Prefetch a certain number of images for rows prior to and subsequent to the currently visible cells
 *
 * @param  tableView   The tableview for which we're going to prefetch images.
 */

- (void)prefetchImagesForTableView:(UITableView *)tableView
{
    NSArray *indexPaths = [self.tableView indexPathsForVisibleRows];
    if ([indexPaths count] == 0) return;

    NSIndexPath *minimumIndexPath = indexPaths[0];
    NSIndexPath *maximumIndexPath = [indexPaths lastObject];

    // they should be sorted already, but if not, update min and max accordingly

    for (NSIndexPath *indexPath in indexPaths) {
        if ([minimumIndexPath compare:indexPath] == NSOrderedDescending)
            minimumIndexPath = indexPath;
        if ([maximumIndexPath compare:indexPath] == NSOrderedAscending)
            maximumIndexPath = indexPath;
    }

    // build array of imageURLs for cells to prefetch

    NSMutableArray *imageURLs = [NSMutableArray array];

    NSArray *precedingRows = [self tableView:tableView indexPathsForPrecedingRows:kPrefetchRowCount fromIndexPath:minimumIndexPath];
    [imageURLs addObjectsFromArray:precedingRows];

    NSArray *followingRows = [self tableView:tableView indexPathsForFollowingRows:kPrefetchRowCount fromIndexPath:maximumIndexPath];
    [imageURLs addObjectsFromArray:followingRows];

    // now prefetch

    if ([imageURLs count] > 0) {
        [[SDWebImagePrefetcher sharedImagePrefetcher] prefetchURLs:imageURLs];
    }
}

/** Retrieve NSIndexPath for a certain number of rows preceding particular NSIndexPath in the table view.
 *
 * @param  tableView  The tableview for which we're going to retrieve indexPaths.
 * @param  count      The number of rows to retrieve
 * @param  indexPath  The indexPath where we're going to start (presumably the first visible indexPath)
 *
 * @return            An array of indexPaths.
 */

- (NSArray *)tableView:(UITableView *)tableView indexPathsForPrecedingRows:(NSInteger)count fromIndexPath:(NSIndexPath *)indexPath
{
    NSMutableArray *indexPaths = [NSMutableArray array];
    NSInteger row = indexPath.row;
    NSInteger section = indexPath.section;

    for (NSInteger i = 0; i < count; i++) {
        if (row == 0) {
            if (section == 0) {
                return indexPaths;
            } else {
                section--;
                row = [tableView numberOfRowsInSection:section] - 1;
            }
        } else {
            row--;
        }
        [indexPaths addObject:[NSIndexPath indexPathForRow:row inSection:section]];
    }

    return indexPaths;
}

/** Retrieve NSIndexPath for a certain number of following particular NSIndexPath in the table view.
 *
 * @param  tableView  The tableview for which we're going to retrieve indexPaths.
 * @param  count      The number of rows to retrieve
 * @param  indexPath  The indexPath where we're going to start (presumably the last visible indexPath)
 *
 * @return            An array of indexPaths.
 */

- (NSArray *)tableView:(UITableView *)tableView indexPathsForFollowingRows:(NSInteger)count fromIndexPath:(NSIndexPath *)indexPath
{
    NSMutableArray *indexPaths = [NSMutableArray array];
    NSInteger row = indexPath.row;
    NSInteger section = indexPath.section;
    NSInteger rowCountForSection = [tableView numberOfRowsInSection:section];

    for (NSInteger i = 0; i < count; i++) {
        row++;
        if (row == rowCountForSection) {
            row = 0;
            section++;
            if (section == [tableView numberOfSections]) {
                return indexPaths;
            }
            rowCountForSection = [tableView numberOfRowsInSection:section];
        }
        [indexPaths addObject:[NSIndexPath indexPathForRow:row inSection:section]];
    }

    return indexPaths;
}
```
