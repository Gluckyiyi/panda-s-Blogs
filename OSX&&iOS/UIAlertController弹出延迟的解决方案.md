# UIAlertController弹出延迟的解决方案

## 前言

最近在做一个小iOS项目的时候，遇到一个小的情况，当时也花了一点精力才得以解决，所以记录一下。当时我在页面的tableview的cell中添加了选中事件，点击的时候弹出一个UIAlertController，但是点击之后，总是要过一会弹出框才会出现，给人一种app很卡的感觉，用户体验很不好。

## 解决方案

### 解决方案一

将```[self presentViewController:alertController animated:YES completion:nil];```

这句代码改成:

```
dispatch_async(dispatch_get_main_queue(), ^{
    
[self presentViewController: alertController animated: YES completion: nil];
});
```

改完之后，点击效果很丝滑。有经验的同学这时候也可以大概知道原因了，延迟弹框的原因和没有及时更新UI有关。

### 解决方案二

一部分小伙伴把**tableViewCell**的**selectionStyle**设成了**UITableViewCellSelectionStyleNone**，只需要将这个设置去掉就好或者设置成**UITableViewCellSelectionStyleDefault**就好；



## 结尾

很简单的一个小问题，但是我觉得可能遇到的人会比较多，希望对大家有帮助吧！(PS:日更真的是很有挑战的，我会尽量准备好的东西和大家分享)