#### 背景
项目技术栈：C#，WPF<br />当前我想要实现点击某个按钮就可以跳转到某个界面，翻阅了项目中的代码，看到了
```csharp
// 按钮事件
private void Btn_Click(object sender, RoutedEventArgs e)
{
    LogBll.Instance.WriteSysLog("xxxxxxxxx");
    NavigationService.Navigate(new Uri("xxxxx.xaml", UriKind.Relative));
    this.TxtSearchBox.Focus(); //一个文本框
}
```
在我的猜测中，我以为是直接在Navigate调用之后就直接进入了对应的页面，然后等页面关闭或者是结束才回到当前这个函数
<br />而实际上，是直接把整个函数执行完，然后再跳转到对应的界面
<br />
#### 疑惑
```csharp
// 1、这行代码还有什么用？
// 2、明明都跳到了B界面了，A界面的组件我还Focus干嘛？
this.TxtSearchBox.Focus(); //一个文本框
```
怀疑Focus操作是为了实现清除文本框文本，防止你输入了文本，然后跳转页面，页面结束后，文本还在<br />（证实为错误，不能实现清除操作）
<br /> 
#### 解答
后来问了下导师，导师说是为了这行代码报异常，无法跳转到新页面，那么驻留在旧页面的时候。旧页面中要求TxtSearchBox组件是要一直保证焦点的，故才在下方插入Focus()操作
```csharp
NavigationService.Navigate(new Uri("xxxxx.xaml", UriKind.Relative));
```
但实际上，我修改了xxxx.xaml改成一个不存在的界面，程序的确可以执行到Focus()操作，这是因为Navigate本来就是执行完整个函数才进行跳转的。Focust()执行了，我的焦点可以聚集吗？不可以，程序直接就报异常退出掉了
<br />
#### 最后
实际上Navigate根本不会在生产环境中报异常，因为Navigate函数出现异常的情况为：
<br />1. 没有为导航目标指定 URI。如果导航目标的URI为null或空，则会抛出ArgumentNullException异常。
2. 导航目标的URI格式不正确。如果导航目标的URI格式不正确，则会抛出UriFormatException异常。
3. 导航目标的XAML文件无法加载。如果导航目标是一个XAML文件，但该文件无法加载，则会抛出XamlParseException异常。
4. 导航目标不是一个有效的Page对象。如果导航目标不是一个有效的Page对象，则会抛出InvalidOperationException异常。
5. 导航目标的构造函数参数不正确。如果导航目标的构造函数需要传递参数，但参数不正确，则会抛出TargetInvocationException异常。
6. 导航目标的代码含有语法错误。如果导航目标的代码含有语法错误，则会抛出XamlParseException异常。
<br />而这些情况，只要测试中不出现，那么生产中就不会出现，这些异常不会随着操作而出现，而是会因代码写错而出现
<br />故Focus()操作是多余的，真异常了，也没用；不异常，也没用，除非你在跳转界面的过程中想要在那个文本框中输入文本，但是，一般不会出现这种情况。最后，新的代码就不加这个操作了，旧的，暂时不管
