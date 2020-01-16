## 问题分析

----------
Fragment的大部分商业场景用法详见 [github地址传送](https://github.com/CharmingGeeker/SignAPP)   

简书地址: http://www.jianshu.com/p/4c5f015b3b6c

一直在看别人的技术贴，今天我也来写点自己的心得！最近在写一个项目用到大量的Fragment后的总结！

我想刚刚接触安卓的或许：

```
FragmentManager     fragmentManager=getSupportFragmentManager();
FragmentTransaction fragmentTransaction=fragmentManager.beginTransaction();
fragmentTransaction.add(ViewId,fragment);// 或者fragmentTransaction.replace(ViewId,fragment);
fragmentTransaction.commit();
```

更好一点的同学用show和hide方法


```
FragmentManager fm = getSupportFragmentManager();
FragmentTransaction ft = fm.beginTransaction();
ft.hide(new FirstFragment())
        .show(new SecondFragment())
        .commit();
```

  诚然这两种都可以切换Fragment，但是面对用户大量点击来回切换，或者你的Fragment本来就很多每次都这样操作那么很快你的应用就会OOM，就算不崩那也会异常的卡顿！so why?

   当我们replace是发生了以下的生命周期：



![这里写图片描述](http://img.blog.csdn.net/20170310001451433?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzU5MTcxNjU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



想想看每次都replace一下！！这世界会有多美好！！！那么问题出在哪？回过头看看代码就会发现每次在add/replace或者show/hide都会new 一个新的实例，这就是致命原因！！！！！

# 废话少说，开始优化

----------

## 方案一：

预加载模式：

```
//首先需要先实例好三个全局Fragment

FragmentManager fm = getSupportFragmentManager();
FragmentTransaction ft = fm.beginTransaction();
ft.add(R.id.fragment, FirstFragment.getInstance());
ft.add(R.id.fragment, SecondFragment.getInstance());
ft.add(R.id.fragment, ThirdFragment.getInstance());
ft.hide(SecondFragment.getInstance());
ft.hide(ThirdFragment.getInstance());
ft.commit();
```

在加载第一个Fragment时就把全部Fragment加载好，下次使用直接调用如：

```
FragmentManager fm = getSupportFragmentManager();
FragmentTransaction ft = fm.beginTransaction();
ft.hide(FirstFragment.getInstance())
        .show(SecondFragment.getInstance())
        .commit();
```

是不是总觉怪怪的，虽然比之前的代码好，但是这种做法很Java，当然需要预加载的朋友依然是不二之选！！！

那有没有更好的方法呢？答案是肯定的

## 方案二：

动态加载模式：



```
//首先需要先实例好n个全局Fragment

//private  Fragment  currentFragment=new Fragment();（全局）

private  FragmentTransaction switchFragment(Fragment targetFragment) {

    FragmentTransaction transaction = getSupportFragmentManager()
            .beginTransaction();
    if (!targetFragment.isAdded()) {
        //第一次使用switchFragment()时currentFragment为null，所以要判断一下
        if (currentFragment != null) {
            transaction.hide(currentFragment);
            }
        transaction.add(R.id.fragment, targetFragment,targetFragment.getClass().getName());

        } else {
            transaction
                    .hide(currentFragment)
                    .show(targetFragment);
                     }
        currentFragment = targetFragment;
       return   transaction;
    }
```


在点击切换Fragment时：

```
@Override
public void onTabSelected(@IdRes int tabId) {

        if (tabId == R.id.tab_one){

            switchFragment(first).commit();

        }
        if (tabId == R.id.tab_two){
            switchFragment(second).commit();
        }
        if (tabId == R.id.tab_three){
            switchFragment(third).commit();
        }
    }
```

现在你的Fragment无论怎么切都不会出现卡顿了，因为你的所有Fragment只会被实例化一次！实例一次的Fragment会被存入内存中，下次切换会判断内存中是否含有要切换的Fragment，如果有就直接复用，没有就add一个新的！优化大法完成！

# 外番
----------
WHAT？等等！只实例一次，那我的Fragment里的数据要更新怎么办？我的回答是——软件关了再次重启！


![这里写图片描述](http://img.blog.csdn.net/20170310001710566?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzU5MTcxNjU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​    


 要是这样，这样的软件真的要逆天了！好在官方提供了onHiddenChanged方法，每次切换hide或者show时该方法会被执行，可以在这里面更新数据！

```
//此方法在Fragment中

@Override
public void onHiddenChanged(boolean hidden) {
    super.onHiddenChanged(hidden);
    if (hidden){
       //Fragment隐藏时调用
    }else {
        //Fragment显示时调用
    }

}
```


此方法是不是比每次add或replace更新数据执行一大坨的生命周期要优雅的多的多！
