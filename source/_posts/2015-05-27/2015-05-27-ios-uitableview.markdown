---
layout: post
title: "UITableview"
date: 2015-05-27 15:48:16 +0800
comments: true
categories: IOS
---
### UITableview
#### UITableView每次滑动一行，类似scroll page enable
    - (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate{  
        if(decelerate) return;  
        [self scrollViewDidEndDecelerating:scrollView];  
    }  
      
    - (void)scrollViewDidEndDecelerating:(UITableView *)tableView {  
        [tableView scrollToRowAtIndexPath:[tableView indexPathForRowAtPoint: CGPointMake(tableView.contentOffset.x, tableView.contentOffset.y+tableView.rowHeight/2)] atScrollPosition:UITableViewScrollPositionTop animated:YES];  
    }  


<!--more-->

#### UITableview系统颜色
    [UIColor groupTableViewBackgroundColor]; 
    [UIColor scrollViewTexturedBackgroundColor];

#### 顶部空白
    self.edgesForExtendedLayout = UIRectEdgeNone;

#### 自适应高度、宽度
    self.tableView.autoresizingMask = UIViewAutoresizingFlexibleWidth |   UIViewiewAutoresizingFlexibleHeight;

#### ios7 tableview separator line dispear bug
    //add tableView:didSelectRowAtIndexPath
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    [tableView reloadRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationAutomatic];

#### 子视图的cell indexpath
- (void)textFieldDidBeginEditing:(UITextField *)textField{
    CGPoint pnt = [self.tableView convertPoint:textField.bounds.origin fromView:textField];
    NSIndexPath* path = [self.tableView indexPathForRowAtPoint:pnt];
    [self.tableView scrollToRowAtIndexPath:path atScrollPosition:UITableViewScrollPositionTop animated:YES];
}

#### 动态cell高度
    - (CGFloat)heightForTextView:(UITextView*)textView containingString:(NSString*)text {
        NSMutableParagraphStyle *paragraphStyle = [[NSMutableParagraphStyle alloc]init];
        paragraphStyle.lineBreakMode = NSLineBreakByWordWrapping;
        NSDictionary *attributes = @{
                                     NSFontAttributeName:[UIFont preferredFontForTextStyle:UIFontTextStyleBody],
                                     NSParagraphStyleAttributeName:paragraphStyle.copy};

        // 计算文本的大小
        CGSize textSize = [text 
                boundingRectWithSize : CGSizeMake(CGRectGetWidth(self.textView.bounds), MAXFLOAT) 
                options : NSStringDrawingUsesLineFragmentOrigin                  attributes : attributes
                context:nil].size;

        return textSize.height + textView.textContainerInset.top + textView.textContainerInset.bottom;
    }

    - (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
        UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"UniqueCellIdentifierForThisUniqueCellLayout"];

        // Configure the cell with content for the given indexPath, for example:
        // cell.textLabel.text = someTextForThisCell;
        // ...

        [cell.contentView setNeedsLayout];
        [cell.contentView layoutIfNeeded];
        CGFloat height = [cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize].height;
        return height;
    }

    - (CGFloat)tableView:(UITableView *)tableView estimatedHeightForRowAtIndexPath:(NSIndexPath *)indexPath {
        // Return a fixed constant if possible, or do some minimal calculations if needed to be able to return an
        // estimated row height that's at least within an order of magnitude of the actual height.
        // For example:
        if ([self isLargeTypeForCellAtIndexPath:indexPath]) {
            return 150.0f;
        } else {
            return 60.0f;
        }
    }

#### UITableview color
##### 1.tableview cell 颜色
    //1.系统默认的颜色设置
    //无色
    cell.selectionStyle = UITableViewCellSelectionStyleNone;
    //蓝色
    cell.selectionStyle = UITableViewCellSelectionStyleBlue;
    //灰色
    cell.selectionStyle = UITableViewCellSelectionStyleGray;

##### 2.自定义颜色和背景设置
    cell.selectedBackgroundView = [[[UIView alloc] initWithFrame:cell.frame] autorelease];
    cell.selectedBackgroundView.backgroundColor = [UIColor xxxxxx];

##### 3自定义UITableViewCell选中时背景
    cell.selectedBackgroundView = [[UIImageView alloc] initWithImage:[UIImage imageNamed:@"cellart.png"]];
    //还有字体颜色
    cell.textLabel.highlightedTextColor = [UIColor xxxcolor];  [cell.textLabel setTextColor:color];//设置cell的字体的颜色

##### 4.设置tableViewCell间的分割线的颜色
    [theTableView setSeparatorColor:[UIColor xxxx ]];

##### 5.设置cell中字体的颜色
    // Customize the appearance of table view cells.
    - (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
    {
      if(0 == indexPath.row)
      {
        cell.textLabel.textColor = ...;
        cell.textLabel.highlightedTextColor = ...;
      }
      ...
    }
