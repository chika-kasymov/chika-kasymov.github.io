---
title:  "Getting the frame of UIBarButtonItem"
date:   2017-09-21 00:00:00
description: Small UIViewController extension to get the frame of UIBarButtonItem instance.
categories:
- UIBarButtonItem
permalink: getting_the_frame_of_uibarbuttonitem
comments: true
---

In this post, I want to share my experience of finding the frame of `UIBarButtonItem`.

You probably already know that unfortunately `UIBarButtonItem` doesn't subclass the `UIView` class and doesn't contain `frame` property. There are some [historical reasons](https://ashfurrow.com/blog/exploring-uibarbuttonitem/#historical-reasons) for that. Therefore developers who need to get frame of `UIBarButtonItem` are using different hacks or ways to accomplish this task.

I couldn't find the one solution which fully solves my problems. Therefore in the below paragraphs, you can find my approach for this uncommon issue and which is working on **iOS 11**.

`UIViewController` extension which I used to get the frame of `UIBarButtonItem` in `UINavigationBar` or `UITolbar`:

``` objc
/* In UIViewController+UIBarButtonItemFrame.h */

#import <UIKit/UIKit.h>

@interface UIViewController (UIBarButtonItemFrame)

- (CGRect)frameForNavBarItem:(UIBarButtonItem *)item;
- (CGRect)frameForToolbarItem:(UIBarButtonItem *)item flexibleSpaceItem:(UIBarButtonItem *)flexibleSpaceItem;

@end


/* In UIViewController+UIBarButtonItemFrame.m */

#import "UIViewController+UIBarButtonItemFrame.h"

@implementation UIViewController (Extensions)

- (CGRect)frameForView:(UIView *)view {
    return [view convertRect:view.bounds toView:nil];;
}

- (void)buttonsForView:(UIView *)view buttons:(NSMutableArray<UIView *> *)buttons {
    for (UIView *subview in view.subviews) {
        if ([subview isKindOfClass:[UIControl class]]) {
            [buttons addObject:subview];
        } else {
            [self buttonsForView:subview buttons:buttons];
        }
    }
}

- (CGRect)frameForNavBarItem:(UIBarButtonItem *)item {
    NSMutableArray *buttons = [[NSMutableArray alloc] init];
    [self buttonsForView:self.navigationController.navigationBar buttons:buttons];

    NSMutableArray *items = [NSMutableArray new];
    [items addObjectsFromArray:self.navigationItem.leftBarButtonItems];
    [items addObjectsFromArray:[[self.navigationItem.rightBarButtonItems reverseObjectEnumerator] allObjects]];

    NSUInteger index = [items indexOfObject:item];
    if (index < buttons.count) {
        UIView *view = buttons[index];
        return [self frameForView:view];
    }

    return [self frameForView:self.navigationController.navigationBar];
}

- (CGRect)frameForToolbarItem:(UIBarButtonItem *)item flexibleSpaceItem:(UIBarButtonItem *)flexibleSpaceItem {
    NSMutableArray *buttons = [[NSMutableArray alloc] init];
    [self buttonsForView:self.navigationController.toolbar buttons:buttons];

    NSMutableArray *toolbarItems = [NSMutableArray arrayWithArray:self.toolbarItems];
    NSMutableArray *itemsToRemove = [NSMutableArray new];
    for (UIBarButtonItem *barButtonItem in toolbarItems) {
        if (barButtonItem == flexibleSpaceItem) {
            [itemsToRemove addObject:barButtonItem];
        }
    }
    [toolbarItems removeObjectsInArray:itemsToRemove];

    NSUInteger index = [toolbarItems indexOfObject:item];
    if (index < buttons.count) {
        UIView *view = buttons[index];
        return [self frameForView:view];
    }

    return [self frameForView:self.navigationController.toolbar];
}

@end
```

1. `- (CGRect)frameForView:(UIView *)view;` - This method is straightforward. It returns the frame of `UIView` in device screen space.
2. `- (void)buttonsForView:(UIView *)view buttons:(NSMutableArray<UIView *> *)buttons;` - This method takes `UINavigationBar` or `UIToolbar` instance as a parameter and then iterates through their subviews until finds all subviews of `UIControl` subclass. In other words, this method will find all views of `UIBarButtonItem`s.
3. `- (CGRect)frameForNavBarItem:(UIBarButtonItem *)item` - This method takes a parameter of type `UIBarButtonItem`, then calls our second method to get all views of `UIBarButtonItem`s of `UINavigationBar`. After the method combines `leftBarButtonItems` and `rightBarButtonItems` to one array. Note that `rightBarButtonItems` are reversed, this decision was made after testing. Then the method simply tries to get the index of input `UIBarButtonItem` instance in the combined array, and if it is found and the index is smaller than the size of the array of views of `UIBarButtonItem`s in `UINavigationBar` the method returns the corresponding view. Otherwise, the frame of whole `UINavigationBar` is returned.
4. `- (CGRect)frameForToolbarItem:(UIBarButtonItem *)item flexibleSpaceItem:(UIBarButtonItem *)flexibleSpaceItem` - This method is almost identical to 3rd one. The difference is that it takes `UIToolbar` to find views of `UIBarButtonItem`s, and also removes unneeded `UIBarButtonItem`s from `toolbarItems`. In my case, it was `UIBarButtonItem` with the type of `UIBarButtonSystemItemFlexibleSpace`. You can have another one(s).

Another thing to note is that I had a logic where `UINavigationBar` or `UIToolbar` items were changed over time. When I called the above methods after changing `UIBarButtonItem` list I noticed that the views were not created immediately (or maybe had zero frames). Therefore in that scenario, you will want to call methods to update the layout of `UINavigationBar` or `UIToolbar` before calling above methods. For example, like this:
``` objc
[self setToolbarItems:SOME_ITEMS animated:false];

[self.navigationController.toolbar setNeedsDisplay];
[self.navigationController.toolbar setNeedsLayout];
[self.navigationController.toolbar layoutIfNeeded];

// then get frames
```

So, that was a description of my solution. It works nicely to solve my problems. I hope that solution will give you an idea of how to approach the problem of finding `UIBarButtonItem` frame.
