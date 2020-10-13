---
title:  "UITextField & Auto Suggestion"
date:   2017-01-29 00:00:00
description: Tutorial on making simple auto suggestion feature for UITextField.
categories:
- UITextField
- Auto Suggestion
- Protocols
permalink: uitextfield_and_auto_suggestion
comments: true
---

One of the time-consuming parts of any application for users is text fields. To save invaluable time for users it's often a good idea to provide helper tools for them. In this article, I gonna show you how to make auto suggestion feature for `UITextField`. For instance, we can use it as the additional feature for the search field.

{% include toc %}

## Quick Introduction

To interest you, that's how the final project will look like:

![Final auto suggestion feature](/assets/images/auto_suggestion.gif)

In the beginning, let's define what our auto suggestion feature will do:

* The auto suggestion will appear and disappear when user will begin and end typing to `UITextField` accordingly. 
* Data to be shown as suggestions will be taken from a local source or by the network.
* Communication between auto suggestion and `UIViewController` will be through a protocol.
* UI will consist from:
	* `UIView` container for `UITableView` to set rounded corners on two sides instead all four sides
	* `UITableView` to show the list of suggestions
	* `UIActivityIndicatorView` to show loading state, if we need to make network connection
	* Two `UIView` for dim effect (One to darken and disable other parts of the screen, second for loading state of `UITableView`)

## Implementation

Let's create the new category for `UITextField` called **AutoSuggestion**. This will make our solution for this problem more flexible. Alternatively, you can create a subclass of `UITextField` if you think auto suggestion feature is too big for a category.

``` objc
//  UITextField+AutoSuggestion.h

#import <UIKit/UIKit.h>

@interface UITextField (AutoSuggestion)
@end

//  UITextField+AutoSuggestion.m

#import "UITextField+AutoSuggestion.h"
@implementation UITextField (AutoSuggestion)
@end
```

Next, we need to add some properties. By default in Objective-C categories can not contain properties, but we can achieve that by using `<objc/runtime.h>` library. It creates properties on runtime, Captain Obvious :)

``` objc
// inside @interface in UITextField+AutoSuggestion.h
@interface UITextField (AutoSuggestion)

@property (nonatomic, strong) UIView *tableContainerView; // 1
@property (nonatomic, strong) UITableView *tableView; // 2
@property (nonatomic, strong) UIView *tableAlphaView; // 3
@property (nonatomic, strong) UIActivityIndicatorView *spinner; // 4
@property (nonatomic, strong) UIView *alphaView; // 5
@property BOOL autoSuggestionIsShowing; // 6
@property CGRect textFieldRectOnWindow; // 7
@property CGRect keyboardFrameBeginRect; // 8
@property (nonatomic, strong) NSString *fieldIdentifier; // 9
@property NSInteger maxNumberOfRows; // 10

@end
```

Explanation:

1. `tableContainerView` is a view to hold our `UITableView`. We will set rounded corners on two sides instead of four by applying the mask to view using `UIBezierPath`. If we set rounded corners by this strategy to `UITableView` instead than part of `UITableView` will not be seen while scrolling.
2. This is our `UITableView` to present data.
3. `tableAlphaView` will be used to show loading state effect.
4. `spinner` also will be used to show loading state effect.
5. `alphaView` is a view to darken and disable outer parts of `UITextField` and suggestion view while editing.
6. `autoSuggestionIsShowing` is a flag to store current state of suggestion view.
7. `textFieldRectOnWindow` is a `CGRect` of `UITextField` where to show suggestion view. Needed to calculate size and position of suggestion view.
8. `keyboardFrameBeginRect` is a `CGRect` of keyboard. Needed to calculate size and position of suggestion view.
9. `fieldIdentifier` helper variable to differentiate fields.
10. `maxNumberOfRows` is a variable to provide maximum number of rows to show in suggestion view instead of default number.

Now to finish we need to provide getters and setters using `<objc/runtime.h>` which will be used at runtime. Below are few examples how it can be achieved for variables of type `BOOL`, `CGRect`, and `UITableView`:

``` objc
// in UITextField+AutoSuggestion.m
- (BOOL)autoSuggestionIsShowing {
    return [objc_getAssociatedObject(self, @selector(autoSuggestionIsShowing)) boolValue];
}

- (void)setAutoSuggestionIsShowing:(BOOL)autoSuggestionIsShowing {
    objc_setAssociatedObject(self, @selector(autoSuggestionIsShowing), @(autoSuggestionIsShowing), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (UITableView *)tableView {
    return objc_getAssociatedObject(self, @selector(tableView));
}

- (void)setTableView:(UITableView *)tableView {
    objc_setAssociatedObject(self, @selector(tableView), tableView, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (CGRect)textFieldRectOnWindow {
    NSValue *textFieldRectOnWindowValue = (NSValue *)objc_getAssociatedObject(self, &textFieldRectOnWindowKey);
    
    if (textFieldRectOnWindowValue != nil) {
        return [textFieldRectOnWindowValue CGRectValue];
    } else {
        return CGRectZero;
    }
}

- (void)setTextFieldRectOnWindow:(CGRect)textFieldRectOnWindow {
    NSValue *textFieldRectOnWindowValue = [NSValue valueWithCGRect:textFieldRectOnWindow];
    objc_setAssociatedObject(self, &textFieldRectOnWindowKey, textFieldRectOnWindowValue, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
```

As you can see from above code snippet, we use two methods: 

1. `objc_setAssociatedObject(_:_:_:_:)`
	
	From documentation:
	> Sets an associated value for a given object using a given key and association policy.
2. `objc_getAssociatedObject(_:_:)`.

	From documentation:
	> Returns the value associated with a given object for a given key.

From my knowledge almost all type of variables can be stored and retrieved using these two methods.

Other properties implementation is straightforward, you can use above examples and try yourself.

Next steps are to present suggestion view. Therefore we need to calculate position and size for views. Since it heavily depends on keyboard frame let's first of all create method to get keybord height. We will use observer on `UIKeyboardDidShowNotification` notification to get keyboard frame.

``` objc
// in UITextField+AutoSuggestion.h
@interface UITextField (AutoSuggestion)
// previous stuff
- (void)observeTextFieldChanges; // 1
@end

// in UITextField+AutoSuggestion.m
// 2
- (void)observeTextFieldChanges {
[[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(getKeyboardHeight:)
                                                 name:UIKeyboardDidShowNotification
                                               object:nil];
}

// 3
- (void)getKeyboardHeight:(NSNotification*)notification {
    NSDictionary* keyboardInfo = [notification userInfo];
    NSValue* keyboardFrameBegin = [keyboardInfo valueForKey:UIKeyboardFrameBeginUserInfoKey];
    self.keyboardFrameBeginRect = [keyboardFrameBegin CGRectValue];
}
```
Explanation:

1. We define `observeTextFieldChanges()` method in the header file because we need to start observing changes in `UITextField` from other places.
2. Also, this method will be extended in next steps. For now, it only contains the line to observe changes for `UIKeyboardDidShowNotification` notifications.
3. To get the frame of the keyboard we need to get its value from the dictionary by key named `UIKeyboardFrameBeginUserInfoKey`.

Next step is to implement the logic of showing and hiding suggestions view.

``` objc
// in UITextField+AutoSuggestion.m

// 1
- (void)observeTextFieldChanges {
	// previous stuff
	[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(toggleAutoSuggestion:) name:UITextFieldTextDidChangeNotification object:self];
	[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(hideAutoSuggestion) name:UITextFieldTextDidEndEditingNotification object:self];
}

// 2
- (void)createSuggestionView {
    UIWindow *appDelegateWindow = [UIApplication sharedApplication].keyWindow;
    self.textFieldRectOnWindow = [self convertRect:self.bounds toView:nil];
    
    if (!self.tableContainerView) {
        self.tableContainerView = [UIView new];
        self.tableContainerView.backgroundColor = [UIColor whiteColor];
    }
    
    if (!self.tableView) {
        self.tableView = [[UITableView alloc] initWithFrame:self.textFieldRectOnWindow style:UITableViewStylePlain];
        self.tableView.tableFooterView = [[UIView alloc] initWithFrame:CGRectZero];
        self.tableView.delegate = self;
        self.tableView.dataSource = self;
        self.tableView.tableFooterView = [UIView new];
    }
    
    if (!self.alphaView) {
        self.alphaView = [[UIView alloc] initWithFrame:appDelegateWindow.bounds];
        self.alphaView.userInteractionEnabled = true;
        self.alphaView.backgroundColor = [[UIColor blackColor] colorWithAlphaComponent:0.5];
        [appDelegateWindow addSubview:self.alphaView];
    }
    
    self.tableView.frame = self.textFieldRectOnWindow;
    [self.tableContainerView addSubview:self.tableView];
    self.tableContainerView.frame = self.textFieldRectOnWindow;
    [appDelegateWindow addSubview:self.tableContainerView];
}

// 3
- (void)showAutoSuggestion {
    if (!self.autoSuggestionIsShowing) {
        [self createSuggestionView];
        self.autoSuggestionIsShowing = YES;
    }
    
    [self reloadContents];
}

// 4
- (void)hideAutoSuggestion {
    if (self.autoSuggestionIsShowing) {
        
        [self.alphaView removeFromSuperview];
        self.alphaView = nil;
        
        [self.tableView removeFromSuperview];
        self.tableView = nil;
        
        [self.tableContainerView removeFromSuperview];
        self.tableContainerView = nil;
        
        self.autoSuggestionIsShowing = NO;
    }
}

// 5
- (void)toggleAutoSuggestion:(NSNotification *)notification {
    if (self.text.length > 0) {
        [self showAutoSuggestion];
    } else {
        [self hideAutoSuggestion];
    }
}

// 6 
- (void)reloadContents {
    [self updateHeight];
    [self updateCornerRadius];
    [self checkForEmptyState];
    
    [self.tableView reloadData];
}

// 7
- (void)updateHeight {
    NSInteger numberOfResults = [self tableView:self.tableView numberOfRowsInSection:0];
    NSInteger maxRowsToShow = self.maxNumberOfRows != 0 ? self.maxNumberOfRows : DEFAULT_MAX_NUM_OF_ROWS;
    CGFloat cellHeight = DEFAULT_ROW_HEIGHT;
    if ([self.tableView numberOfRowsInSection:0] > 0) {
        cellHeight = [self tableView:self.tableView heightForRowAtIndexPath:[NSIndexPath indexPathForRow:0 inSection:0]];
    }
    CGFloat height = MIN(maxRowsToShow, numberOfResults) * cellHeight; // check if numberOfResults < maxRowsToShow
    height = MAX(height, cellHeight); // if 0 results, set height = `cellHeight`
    
    CGRect frame = self.textFieldRectOnWindow;
    
    if ([self showSuggestionViewBelow]) {
        CGFloat maxHeight = [UIScreen mainScreen].bounds.size.height - (frame.origin.y + frame.size.height) - INSET - self.keyboardFrameBeginRect.size.height; // max possible height
        height = MIN(height, maxHeight); // set height = `maxHeight` if it's smaller than current `height`
        
        frame.origin.y += frame.size.height;
    } else {
        CGFloat maxHeight = frame.origin.y - INSET;  // max possible height
        height = MIN(height, maxHeight); // set height = `maxHeight` if it's smaller than current `height`
        
        frame.origin.y -= height;
    }
    
    frame.size.height = height;
    self.tableView.frame = CGRectMake(0, 0, frame.size.width, frame.size.height);
    self.tableContainerView.frame = frame;
}

// 8
- (void)updateCornerRadius {
    // code snippet from SO answer (http://stackoverflow.com/a/13163693/1760199)
    UIRectCorner corners = (UIRectCornerBottomLeft | UIRectCornerBottomRight);
    if (![self showSuggestionViewBelow]) {
        corners = (UIRectCornerTopLeft | UIRectCornerTopRight);
    }
    
    UIBezierPath *maskPath = [UIBezierPath
                              bezierPathWithRoundedRect:self.tableContainerView.bounds
                              byRoundingCorners:corners
                              cornerRadii:CGSizeMake(6, 6)
                              ];
    
    CAShapeLayer *maskLayer = [CAShapeLayer layer];
    
    maskLayer.frame = self.bounds;
    maskLayer.path = maskPath.CGPath;
    
    self.tableContainerView.layer.mask = maskLayer;
}

// 9
- (void)checkForEmptyState {
    if ([self tableView:self.tableView numberOfRowsInSection:0] == 0) {
        UILabel *emptyTableLabel = [[UILabel alloc] initWithFrame:self.tableView.bounds];
        emptyTableLabel.textAlignment = NSTextAlignmentCenter;
        emptyTableLabel.font = [UIFont systemFontOfSize:16];
        emptyTableLabel.textColor = [UIColor grayColor];
        emptyTableLabel.text = @"No matches";
        self.tableView.backgroundView = emptyTableLabel;
    } else {
        self.tableView.backgroundView = nil;
    }
}

// 10
- (BOOL)showSuggestionViewBelow {
    CGRect frame = self.textFieldRectOnWindow;
    return frame.origin.y + frame.size.height/2.0 < ([UIScreen mainScreen].bounds.size.height - self.keyboardFrameBeginRect.size.height)/2.0;
}
```

Explanation:

1. Add extra observers to handle `UITextField` changes.
2. In this method, we create all base views and setting initial frames.
3. Method to show suggestion view.
4. Method to hide suggestion view.
5. Use above two methods to toggle between states.
6. This method calls other methods to recalculate suggestion view frame, update corners and set text if data is empty.
7. In this method, we decide where suggestion view should be shown (above or below) and also calculate the size.
8. This method used to set corners on top or bottom regarding the position of suggestion view.
9. Helper method to set text if data is empty.
10. Another helper method to decide where to show suggestion view.

Sorry, if too many lines of code are present. I've very little experience in writing tutorials. I hope you will understand and forgive me :).

Now we need to implement `UITableViewDatasource` and `UITableViewDelegate` protocols but before let's define our own. It will be used to interact with outer classes. Data to be shown will be passed through the protocol and also it will notify when something changed.

``` objc
// in UITextField+AutoSuggestion.h
@protocol UITextFieldAutoSuggestionDataSource <NSObject>
- (UITableViewCell *)autoSuggestionField:(UITextField *)field tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath forText:(NSString *)text; // 1
- (NSInteger)autoSuggestionField:(UITextField *)field tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section forText:(NSString *)text; // 2
@optional
- (void)autoSuggestionField:(UITextField *)field textChanged:(NSString *)text; // 3
- (CGFloat)autoSuggestionField:(UITextField *)field tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath forText:(NSString *)text; // 4
- (void)autoSuggestionField:(UITextField *)field tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath forText:(NSString *)text; // 5
@end

@interface UITextField (AutoSuggestion)
// previous stuff
@property (weak, nonatomic) id<UITextFieldAutoSuggestionDataSource> autoSuggestionDataSource; // 6
@end

// in UITextField+AutoSuggestion.m
// 7
- (id<UITextFieldAutoSuggestionDataSource>)autoSuggestionDataSource {
    return objc_getAssociatedObject(self, @selector(autoSuggestionDataSource));
}

- (void)setAutoSuggestionDataSource:(id<UITextFieldAutoSuggestionDataSource>)autoSuggestionDataSource {
    objc_setAssociatedObject(self, @selector(autoSuggestionDataSource), autoSuggestionDataSource, OBJC_ASSOCIATION_ASSIGN);
}
```

Explanation:

1. Using this method developers will have an opportunity to provide their custom designed cells.
2. This method will ask the number of suggestions to show.
3. This and below two methods are optional. We can use this method to handle field changes.
4. Custom height for `UITableViewCell`. By default, height is 44 points.
5. Do something if the cell was selected.
6. This is a variable to store our data source.
7. Again we must define getter and setter for the new property.

Let's now implement `UITableViewDatasource` and `UITableViewDelegate` methods combining with our own protocol.

``` objc
// in UITextField+AutoSuggestion.h
// 1
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {
    return 1;
}

// 2
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    BOOL implementsDatasource = self.autoSuggestionDataSource && [self.autoSuggestionDataSource respondsToSelector:@selector(autoSuggestionField:tableView:numberOfRowsInSection:forText:)];
    NSAssert(implementsDatasource, @"UITextField must implement data source before using auto suggestion.");
    
    return [self.autoSuggestionDataSource autoSuggestionField:self tableView:tableView numberOfRowsInSection:section forText:self.text];
}

// 3
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    BOOL implementsDatasource = self.autoSuggestionDataSource && [self.autoSuggestionDataSource respondsToSelector:@selector(autoSuggestionField:tableView:cellForRowAtIndexPath:forText:)];
    NSAssert(implementsDatasource, @"UITextField must implement data source before using auto suggestion.");
                                                
    return [self.autoSuggestionDataSource autoSuggestionField:self tableView:tableView cellForRowAtIndexPath:indexPath forText:self.text];
}

// 4
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    if (self.autoSuggestionDataSource && [self.autoSuggestionDataSource respondsToSelector:@selector(autoSuggestionField:tableView:heightForRowAtIndexPath:forText:)]) {
        [self.autoSuggestionDataSource autoSuggestionField:self tableView:tableView heightForRowAtIndexPath:indexPath forText:self.text];
    }
    
    return DEFAULT_ROW_HEIGHT;
}

// 5
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    if (self.autoSuggestionDataSource && [self.autoSuggestionDataSource respondsToSelector:@selector(autoSuggestionField:tableView:didSelectRowAtIndexPath:forText:)]) {
        [self.autoSuggestionDataSource autoSuggestionField:self tableView:tableView didSelectRowAtIndexPath:indexPath forText:self.text];
    }
    
    [self hideAutoSuggestion];
}

// 6
- (void)toggleAutoSuggestion:(NSNotification *)notification {
    if (self.text.length > 0) {
        [self showAutoSuggestion];
        
        if ([self.autoSuggestionDataSource respondsToSelector:@selector(autoSuggestionField:textChanged:)]) {
            [self.autoSuggestionDataSource autoSuggestionField:self textChanged:self.text];
        }
    } else {
        [self hideAutoSuggestion];
    }
}
```

Explanation:

1. The number of sections always will be 1.
2. The number of rows will be asked from `UITextField` data source we created. If not implemented, assert will stop the application and show the error message.
3. The same as above but instead this method will ask for `UITableViewCell`.
4. Height for cells. By default is `DEFAULT_ROW_HEIGHT` which is equal to 44 points.
5. This method will hide auto suggestion view and call data source method if implemented.
6. In above code snippets, we already implemented this method. Now we extend it by adding our data source to it. Data source method will be fired if implemented.

And the last step is to add loading state.

``` objc
// in UITextField+AutoSuggestion.h
- (void)setLoading:(BOOL)loading;

// in UITextField+AutoSuggestion.m
- (void)setLoading:(BOOL)loading {
    if (loading) {
        if (!self.tableAlphaView) {
            self.tableAlphaView = [[UIView alloc] initWithFrame:self.tableView.bounds];
            self.tableAlphaView.backgroundColor = [[UIColor whiteColor] colorWithAlphaComponent:0.8];
            [self.tableView addSubview:self.tableAlphaView];
            
            self.spinner = [[UIActivityIndicatorView alloc] initWithFrame:CGRectMake(0, 0, 24, 24)];
            self.spinner.center = self.tableAlphaView.center;
            self.spinner.color = [UIColor blackColor];
            [self.tableAlphaView addSubview:self.spinner];
            
            [self.spinner startAnimating];
        }
    } else {
        if (self.tableAlphaView) {
            [self.spinner startAnimating];
            
            [self.spinner removeFromSuperview];
            self.spinner = nil;
            
            [self.tableAlphaView removeFromSuperview];
            self.tableAlphaView = nil;
        }
    }
}
```

To use this category we should specify which class will conform to the data source and implement its methods.

For example in demo project it looks like this:

``` objc
// 1
@interface ViewController () </* other protocols */, UITextFieldAutoSuggestionDataSource>
@end

@implementation ViewController

// in `viewDidLoad:` or any other method where views are loaded
// 2
- (void)viewDidLoad {
	// other stuff
	self.textField1.autoSuggestionDataSource = self;
    self.textField1.fieldIdentifier = @"FIELD_ID";
    [self.textField1 observeTextFieldChanges];
}

@end

#pragma mark - UITextFieldAutoSuggestionDataSource

// 3
- (UITableViewCell *)autoSuggestionField:(UITextField *)field tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath forText:(NSString *)text {
	static NSString *cellIdentifier = @"AutoSuggestionCell";
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:cellIdentifier];
    
    if (!cell) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:cellIdentifier];
    }
    
    // configure cell and fill with some data
    cell.textLabel.text = @"some data";
    
    return cell;
}

// 4
- (NSInteger)autoSuggestionField:(UITextField *)field tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section forText:(NSString *)text {
	return DATA.count; // where DATA is your suggestion model
}

// 5
- (CGFloat)autoSuggestionField:(UITextField *)field tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath forText:(NSString *)text {
    return 50; // as an example
}

// 6
- (void)autoSuggestionField:(UITextField *)field tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath forText:(NSString *)text {
    self.textField2.text = DATA.count[indexPath.row];
}
```

Explanation:

1. Conforming to the protocol of our category.
2. Setting which field should have auto suggestion feature and calling `observeTextFieldChanges()` method to start observing changes.

Steps **3**, **4**, **5** and **6** and just implementations of data source methods.

That's it! It was good experience writing our own `UITextField` category.

## Conclusion

As we can see categories are the powerful feature of the Objective-C language. By using it we can extend the functionality of any base API class.

But they should be used carefully. For example, when I first implemented auto suggestion feature I implemented the `dealloc` method. After that, strange memory crashes occurred. Which means - do not override `dealloc` method in categories.

Thank you for your attention!

**PS:** Full code can be found at this [link](https://github.com/chika-kasymov/UITextField_AutoSuggestion).