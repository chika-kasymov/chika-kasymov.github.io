---
title:  "Common issues of UIStackView"
date:   2016-12-01 00:00:00
description: Small overview of common problems and issues that arise when using UIStackView.
categories:
- UIStackView
permalink: common-issues-of-uistackview
comments: true
---

I want to share my experience using of `UIStackView` in my last projects. This post will be about common issues and problems that arise when using `UIStackView`.

<div class="notice--info"><b>Note:</b> Most of the solutions to problems I met can be found in Stack Overflow, blogs or other web resources. My aim is to collect solutions for frequent issues.</div>

{% include toc %}

## NSLayoutConstraint Warnings

Most of the developers using `UIStackView` for its flexibility. You can add or insert view to a specific position, remove needed views or just make view hidden without extra work. `UIStackView` has good API, but sometimes developers should do some preprocessing to avoid unneeded behavior and console warnings.

Often I needed to add views to `UIStackView` in particular order and then just make some views hidden or not hidden. For example, to show error message below `UITextField` or disclaimer with some text.

You may saw some warnings when you make some arranged subview of `UIStackView` hidden. In most cases, it happens because your view has subviews, and positions are specified by Auto Layout.

This is an example warning:

```
Probably at least one of the constraints in the following list is one you don't want. Try this: (1) look at each constraint and try to figure out which you don't expect; (2) find the code that added the unwanted constraint or constraints and fix it. (Note: If you're seeing NSAutoresizingMaskLayoutConstraints that you don't understand, refer to the documentation for the UIView property translatesAutoresizingMaskIntoConstraints) 
(
    "<NSLayoutConstraint:0x7fa3a5004310 V:[App.DummyView:0x7fa3a5003fd0(40)]>",
    "<NSLayoutConstraint:0x7fa3a3e44190 'UISV-hiding' V:[App.DummyView:0x7fa3a5003fd0(0)]>"
)

```

To avoid these warnings you should change the priority of your `NSLayoutConstraint`'s from **1000** (required) to **999**. 

``` objc
self.labelHeightConstraint.priority = 999;
```

You can ask me, for which constraints? It depends. If you are using vertical `UIStackView` you should reduce priorities of your height, top, and bottom constraints. In the other hand for horizontal `UIStackView`, you need to reduce priorities of width, left and right constraints. In some cases, all constraint priorities should be reduced.

Therefore you can add a category for `UIView` class for convenience. For example:

``` objc
/* In .h file */
#import <UIKit/UIKit.h>

@interface UIView (Extensions)

/**
 Method to set UILayoutPriority for all UIView's constraints.
 Purpose of this method is to avoid warnings when using view inside UIStackView and
 calling methods hidden = true/false.
 @param priority UILayoutPriority value
 */
- (void)setAllConstraintsPriorityTo:(UILayoutPriority)priority;

@end

/* In .m file */
#import "UIView+Extensions.h"

@implementation UIView (Extensions)

- (void)setAllConstraintsPriorityTo:(UILayoutPriority)priority {
    for (NSLayoutConstraint *constraint in self.constraints) {
        constraint.priority = priority;
    }
}

@end
```

Above solution will resolve many warnings, but we also have the special case. It will be described in the below section.

## Nested UIStackView

Sometimes we need to combine multiple `UIStackView`. For example, use horizontal one inside of vertical and in other scenarios. 

To avoid `NSLayoutConstraint` warnings you need to wrap your child `UIStackView` with `UIView`. And do not forget to reduce constraint priorities. 

Again, for convenience you, can create a subclass of `UIView` with `UIStackView` inside it. For instance:

``` objc
/* In .h file */
#import <UIKit/UIKit.h>

@interface StackedView : UIView
@property (nonatomic, strong, readonly) UIStackView *stackView;
@end

/* In .m file */
#import "StackedView.h"
#import "UIView+Extensions.h"
#import "PureLayout.h"

@implementation StackedView {

- (instancetype)init {
    self = [super init];
    
    if (self) {
        [self setup];
    }
    
    return self;
}

- (instancetype)initWithCoder:(NSCoder *)aDecoder {
    self = [super initWithCoder:aDecoder];
    
    if (self) {
        [self setup];
    }
    
    return self;
}

- (void)setup {
    _stackView = [UIStackView newAutolayoutView];
    self.stackView.alignment = UIStackViewAlignmentFill;
    self.stackView.distribution = UIStackViewDistributionFill;
    self.stackView.axis = UILayoutConstraintAxisVertical;
    [self addSubview:self.stackView];
    [self.stackView autoPinEdgesToSuperviewEdges];
    
    [self setAllConstraintsPriorityTo:999];
}

@end
```

That's it about `NSLayoutConstraint` warnings which appear when using `UIStackView`.

## UIStackView inside UIScrollView

Since the size of device screens are limited `UIStackView` is useless if we have big content. You may correctly guess that this problem can be solved by putting stack view inside `UIScrollView`.

There are a number of approaches to do this. This is my favorite:

``` objc
/* In .h file */
#import <UIKit/UIKit.h>

typedef enum ScrollDirection: NSInteger {
    ScrollDirectionVertical,
    ScrollDirectionHorizontal
} ScrollDirection;

@interface StackedScrollView : UIScrollView

- (instancetype)initWithScrollDirection:(ScrollDirection)scrollDirection;

@property (nonatomic, strong, readonly) UIStackView *stackView;
@property (nonatomic) ScrollDirection scrollDirection;

@end

/* In .m file */
#import "StackedScrollView.h"
#import "UIView+Extensions.h"
#import "PureLayout.h"

@interface StackedScrollView ()
@property (nonatomic, strong) NSMutableArray *stackViewConstraints;
@end

@implementation StackedScrollView

- (instancetype)initWithScrollDirection:(ScrollDirection)scrollDirection {
    self = [super init];
    
    if (self) {
        [self setup];
        
        self.scrollDirection = scrollDirection;
    }
    
    return self;
}

- (void)setup {
    self.stackViewConstraints = [NSMutableArray new];
    self.stackView = [UIStackView new];
    self.stackView.alignment = UIStackViewAlignmentFill;
    self.stackView.distribution = UIStackViewDistributionFill;
    [self addSubview:self.stackView];
}

- (void)layoutSubviews {
    [super layoutSubviews];
    
    self.contentSize = self.stackView.frame.size;
}

#pragma mark - Setters

- (void)setScrollDirection:(ScrollDirection)scrollDirection {
    _scrollDirection = scrollDirection;
    
    if (self.stackViewConstraints) {
        [self.stackViewConstraints autoRemoveConstraints];
    }
    if (self.stackView) {
        if (self.scrollDirection == ScrollDirectionHorizontal) {
            self.stackView.axis = UILayoutConstraintAxisHorizontal;
            
            [self.stackViewConstraints addObject:[self.stackView autoPinEdgeToSuperviewEdge: ALEdgeTop]];
            [self.stackViewConstraints addObject:[self.stackView autoPinEdgeToSuperviewEdge:ALEdgeLeft]];
            [self.stackViewConstraints addObject:[self.stackView autoPinEdgeToSuperviewEdge: ALEdgeBottom]];
        } else {
            self.stackView.axis = UILayoutConstraintAxisVertical;
            
            [self.stackViewConstraints addObject:[self.stackView autoPinEdgeToSuperviewEdge:ALEdgeTop]];
            [self.stackViewConstraints addObject:[self.stackView autoPinEdgeToSuperviewEdge:ALEdgeLeft]];
            [self.stackViewConstraints addObject:[self.stackView autoPinEdgeToSuperviewEdge:ALEdgeRight]];
        }
    }
    
    [self setAllConstraintsPriorityTo:999];
	
    [self setNeedsLayout];
    [self layoutIfNeeded];
}

@end
```

## Removing Arranged Subviews

And the last thing to mention is the proper way to remove arranged subviews from `UIStackView`. It may be seen stupid, but I didn't understand initially why removed arranged subviews by method `removeArrangedSubview:` are still appearing on the screen. Only be reading the documentation of this method I understood that we also have to remove it from view hierarchy by hand.

As Apple documentation says on `removeArrangedSubview:` method:

> This method removes the provided view from the stack’s `arrangedSubviews` array. The view’s position and size will no longer be managed by the stack view. **However, this method does not remove the provided view from the stack’s** `subviews` **array; therefore, the view is still displayed as part of the view hierarchy.**

> To prevent the view from appearing on screen after calling the stack’s `removeArrangedSubview:` method, explicitly remove the view from the subviews array by calling the view’s `removeFromSuperview` method, or set the view’s `hidden` property to `YES`.

Therefore to completely remove your view just call `removeFromSuperview` method.

## Useful links

1. [UIStackView - layout constraint issues when hiding stack views](http://stackoverflow.com/questions/33642888/uistackview-layout-constraint-issues-when-hiding-stack-views)
2. [Is it possible for UIStackView to scroll?](http://stackoverflow.com/questions/31668970/is-it-possible-for-uistackview-to-scroll)
3. [Documentation on `removeArrangedSubview:` method](https://developer.apple.com/reference/uikit/uistackview/1616235-removearrangedsubview?language=objc)