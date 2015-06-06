---
layout: post
title: "ios系统服务"
date: 2015-05-27 15:45:42 +0800
comments: true
categories: IOS
---
#### AddressBook
AddressBook不能使用arc
1. ABAddressBookRef 通讯录对象
2. ABRecordRef  联系人或群组 ABRecordGetRecordType() ABRecordGetRecordID()
3. ABPersonRef
4. ABGroupRef
5. ABPersonCreate():创建一个类型为“kABPersonType”的ABRecordRef
6. ABRecordCopyValue():取得指定属性的值
7. ABRecordCopyCompositeName():取得联系人（或群组）的复合信息（对于联系人则包括：姓、名、公司等信息，对于群组则返回组名称）
8. ABRecordSetValue():设置ABRecordRef的属性值
9. ABMutableMultiValueRef ABMultiValueAddValueAndLabel()方法依次添加属性值
10. ABRecordRemoveValue():删除指定的属性值

<!--more-->

#### 通讯录的访问步骤
1. 调用ABAddressBookCreateWithOptions()方法创建通讯录对象ABAddressBookRef。
2. 调用ABAddressBookRequestAccessWithCompletion()方法获得用户授权访问通讯录
3. 调用ABAddressBookCopyArrayOfAllPeople()、ABAddressBookCopyPeopleWithName()方法查询联系人信息。
4. 读取联系人后如果要显示联系人信息则可以调用ABRecord相关方法读取相应的数据；如果要进行修改联系人信息，则可以使用对应的方法修改ABRecord信息，然后调用ABAddressBookSave()方法提交修改；如果要删除联系人，则可以调用ABAddressBookRemoveRecord()方法删除，然后调用ABAddressBookSave()提交修改操作
5. 也就是说如果要修改或者删除都需要首先查询对应的联系人，然后修改或删除后提交更改。如果用户要增加一个联系人则不用进行查询，直接调用ABPersonCreate()方法创建一个ABRecord然后设置具体的属性，调用ABAddressBookAddRecord方法添加即可



#### 系统应用
1. 打电话：tel:或者tel://、telprompt:或telprompt://(拨打电话前有提示)
2. 发短信：sms:或者sms://
3. 发送邮件：mailto:或者mailto://
4. 启动浏览器：http:或者http://
```
//打电话
- (IBAction)callClicK:(UIButton *)sender {
    NSString *phoneNumber=@"18500138888";
//    NSString *url=[NSString stringWithFormat:@"tel://%@",phoneNumber];//这种方式会直接拨打电话
    NSString *url=[NSString stringWithFormat:@"telprompt://%@",phoneNumber];//这种方式会提示用户确认是否拨打电话
    [self openUrl:url];
}

#pragma mark - 私有方法
-(void)openUrl:(NSString *)urlStr{
    //注意url中包含协议名称，iOS根据协议确定调用哪个应用，例如发送邮件是“sms://”其中“//”可以省略写成“sms:”(其他协议也是如此)
    NSURL *url=[NSURL URLWithString:urlStr];
    UIApplication *application=[UIApplication sharedApplication];
    if(![application canOpenURL:url]){
        NSLog(@"无法打开\"%@\"，请确保此应用已经正确安装.",url);
        return;
    }
    [[UIApplication sharedApplication] openURL:url];
}
```

#### 系统服务
1. 如果想要在应用程序内部完成这些操作则可以利用iOS中的MessageUI.framework,它提供了关于短信和邮件的UI接口供开发者在应用程序内部调用.在MessageUI.framework中主要有两个控制器类分别用于发送短信（MFMessageComposeViewController）和邮件（MFMailComposeViewController）,它们均继承于UINavigationController。由于两个类使用方法十分类似，这里主要介绍一下MFMessageComposeViewController使用步骤：

>1. 创建MFMessageComposeViewController对象。
2.  设置收件人recipients、信息正文body，如果运行商支持主题和附件的话可以设置主题subject、附件attachments（可以通过canSendSubject、canSendAttachments方法判断是否支持）
3. 设置代理MFMessageComposeViewControllerDelegate

```
- (IBAction)sendMessageClick:(UIButton *)sender {
    if([MFMessageComposeViewController canSendText]){
        MFMessageComposeViewController *messageController=[[MFMessageComposeViewController alloc]init];
        //收件人
        messageController.recipients=[self.receivers.text componentsSeparatedByString:@","];
        //信息正文
        messageController.body=self.body.text;
        //设置代理,注意这里不是delegate而是messageComposeDelegate
        messageController.messageComposeDelegate=self;
        //如果运行商支持主题
        if([MFMessageComposeViewController canSendSubject]){
            messageController.subject=self.subject.text;
        }
        //如果运行商支持附件
        if ([MFMessageComposeViewController canSendAttachments]) {
            /*第一种方法*/
            //messageController.attachments=...;
            
            /*第二种方法*/
            NSArray *attachments= [self.attachments.text componentsSeparatedByString:@","];
            if (attachments.count>0) {
                [attachments enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
                    NSString *path=[[NSBundle mainBundle]pathForResource:obj ofType:nil];
                    NSURL *url=[NSURL fileURLWithPath:path];
                    [messageController addAttachmentURL:url withAlternateFilename:obj];
                }];
            }
            
            /*第三种方法*/
//            NSString *path=[[NSBundle mainBundle]pathForResource:@"photo.jpg" ofType:nil];
//            NSURL *url=[NSURL fileURLWithPath:path];
//            NSData *data=[NSData dataWithContentsOfURL:url];
            /**
             *  attatchData:文件数据
             *  uti:统一类型标识，标识具体文件类型，详情查看：帮助文档中System-Declared Uniform Type Identifiers
             *  fileName:展现给用户看的文件名称
             */
//            [messageController addAttachmentData:data typeIdentifier:@"public.image"  filename:@"photo.jpg"];
        }
        [self presentViewController:messageController animated:YES completion:nil];
    }
}

#pragma mark - MFMessageComposeViewController代理方法
//发送完成，不管成功与否
-(void)messageComposeViewController:(MFMessageComposeViewController *)controller didFinishWithResult:(MessageComposeResult)result{
    switch (result) {
        case MessageComposeResultSent:
            NSLog(@"发送成功.");
            break;
        case MessageComposeResultCancelled:
            NSLog(@"取消发送.");
            break;
        default:
            NSLog(@"发送失败.");
            break;
    }
    [self dismissViewControllerAnimated:YES completion:nil];
}
```
