---
layout: post
title: "Using SWRevealViewController Together With RBStoryboardLink"
date: 2015-05-27 15:52:30 +0800
comments: true
categories: IOS
---
```
http://samwize.com/2015/03/26/using-swrevealviewcontroller-together-with-rbstoryboardlink/
```

I posted about how to modularize storyboard using RBStoryboardLink and how to use SWRevealViewController.
But to use both of them together is not straight forward.
This is because both library makes use of custom segues to do what they want to do. And it is not possible for you to customize 1 segue to satisfy both!
To make them work together:
Create the `RBStoryboardLink` suggorate view controller and setup `storyboardName` as per normal, but don’t create any segue to it. It will be alone.
Give it a storyboard ID eg. “surrogate”
Don’t use `SWRevealViewControllerSeguePushController` segue too
Instead, call the revealViewController’s `pushFrontViewController` method manually.
The code:

```
- (IBAction)buttonTapped:(id)sender {
    UIViewController *vc = [self.storyboard instantiateViewControllerWithIdentifier:@"surrogate"];
    [self.revealViewController pushFrontViewController:vc animated:YES];
}
```

If the view controller is not getting behind the status bar correctly, set `needsTopLayoutGuide` key to NO in the suggorate.