---
title: Unity插件制作学习笔记
typora-root-url: Unity插件制作学习笔记
categories: 学习笔记
date: 2023/10/25
tags:
- 程序
- unity
abbrlink: 51902

---

记录自己学习unity制作插件时的细节和tips，有任何问题或疑惑请随时通过主页联系方式联系我一起探讨。
<!--more-->

---

<!-- toc -->

---

# 一、自定义Inspector界面

## 1、属性相关标识

5种变量在inspector隐藏和显示的方法

```c#
using System;

public class Value : MonoBehaviour
{
    
    public int int1;  

    [HideInInspector]//直接隐藏在inspector  
    public int int2=3;  

    [NonSerialized]//隐藏的同时不会保存被inspector序列化后的值   
    public int int3=3;//即更改inspector的值不会影响脚本的初始赋值  

    [SerializeField]//将私有变量显示在inspector  
    private int int4;  

    public ValueTest valueTest = new ValueTest();  

}

[Serializable]//可让自定义的类显示在inspector  
public class ValueTest   
{
    public int int5;  
}
```

最终在inspector的表现效果如下：

![image-20231025225218480](image-20231025225218480.png)

这里涉及到unity中inspector的数据持久化问题，即序列化，参考文档：[脚本序列化 - Unity 手册](https://docs.unity.cn/cn/2020.3/Manual/script-Serialization.html)

“序列化是将数据结构或对象状态转换为 Unity 可存储并在以后可重构的格式的自动过程。”

如果在自定义inspector的脚本中，没有使用派生子editor基类的方法FindProperty()来获取target中的变量时，是无法被序列化保存的，需要注意。



## 2、自定义界面属性

创建自定义界面需要在Asset路径下创建名为Editor的文件夹，此命名的文件夹打包时不会被打包（Resources文件夹同理，在load方法加载资源时要注意路径[（Resources-Load - Unity 脚本 API）](https://docs.unity.cn/cn/2019.3/ScriptReference/Resources.Load.html)）

创建上述Value的自定义inspector：

第一步：创建继承Editor基类的脚本，并将其放在Editor文件夹下，并声明是Value的界面

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;

[CustomEditor(typeof(Value))]
public class ValueEditor :Editor
{
  
    public SerializedObject mObj;   //定义序列化目标对象
    //序列化出来几个属性
    //属性对应的是[CustomEditor(typeof(Value))]中，Value类里面的属性

    public SerializedProperty int1;
    public SerializedProperty int2;
    public SerializedProperty int3;
    public SerializedProperty int4;
    public SerializedProperty valueTest;

    //选择当前游戏对象
    private void OnEnable()
    {
        //初始化
        this.mObj = new SerializedObject(target);
        //通过字符串名字，查找Test1类里面的属性，并进行初始化
        this.int1 = this.mObj.FindProperty("int1");
        this.int2 = this.mObj.FindProperty("int2");
        this.int3 = this.mObj.FindProperty("int3");
        this.int4 = this.mObj.FindProperty("int4");
        this.valueTest = this.mObj.FindProperty("valueTest");
    }

    //绘制
    public override void OnInspectorGUI()
    {
        this.mObj.Update();

        EditorGUILayout.PropertyField(this.int1);
        EditorGUILayout.PropertyField(this.int2);
        //EditorGUILayout.PropertyField(this.int3);
        //因为int3是不被序列化的，所以这里不能写此语句
        EditorGUILayout.PropertyField(this.int4);
        EditorGUILayout.PropertyField(this.valueTest, true);
        //true属性意思是设置显示子节点，也就是显示TypeDemo里面的属性

        this.mObj.ApplyModifiedProperties();
        //应用属性修改
    }
}

```

效果如图

![image-20231025232755437](image-20231025232755437.png)

可以看到自定义的Editor界面不受类本身的属性标识控制。

## 3、自定义序列化属性字段

此时，我想要当前属性随着我的枚举值的变化而变化，即我选择一个枚举时，其他属性处于不被editor绘制的状态。

方法：

第一步，创建并书写对应类的脚本和编辑器脚本，这里分别命名为EnumEditor和Enum。

```c#
using System;

public class Enum : MonoBehaviour
{
    public selectedEnum selectedEnum;
    public GameObject 碰撞盒;
    public GameObject 特效;
    public AnimationClip 动作;
}

[Serializable]
public enum selectedEnum
{ 
碰撞盒,
特效,
动作,
}
```

仅有Enum时

![image-20231025234839268](image-20231025234839268.png)

EnumEditor：

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;

[CustomEditor(typeof(Enum))]
public class EnumEditor : Editor
{
    public SerializedObject mObj;
    public SerializedProperty selectedEnum;
    public SerializedProperty 碰撞盒;
    public SerializedProperty 特效;
    public SerializedProperty 动作;

    private void OnEnable()
    {
        //初始化
        this.mObj = new SerializedObject(target);
        this.selectedEnum = this.mObj.FindProperty("selectedEnum");
        this.碰撞盒 = this.mObj.FindProperty("碰撞盒");
        this.特效 = this.mObj.FindProperty("特效");
        this.动作 = this.mObj.FindProperty("动作");
    }

    public override void OnInspectorGUI()
    {
        this.mObj.Update();
        EditorGUILayout.PropertyField(this.selectedEnum);
        switch (this.selectedEnum.enumValueIndex)
        {
            case 0:
                EditorGUILayout.PropertyField(this.碰撞盒);
                break;
            case 1:
                EditorGUILayout.PropertyField(this.特效);
                break;
            case 2:
                EditorGUILayout.PropertyField(this.动作);
                break;
        }

        this.mObj.ApplyModifiedProperties();
    }
}

```

编写后

![image-20231025235035035](image-20231025235035035.png)

## 4、自定义操作界面

当我们需要自定义操作界面时：

需要用到EditorGUILayout方法：[EditorGUILayout - Unity 脚本 API](https://docs.unity.cn/cn/2019.4/ScriptReference/EditorGUILayout.html)

以下是常用的。

### a、界面的启动和关闭

创建一个类，存放在Assets-->Editor里面，并继承自EditorWindow

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;
public class WindowsEditor : EditorWindow
{
    [MenuItem("Menu/EditorWindow")]
    //声明窗口路径

    public static void ShowWindow()
    {
        //CreateInstance会允许创建多个窗口，推荐使用系统自带的单例模式getwindows
        //使用官方提供的实例化窗口方法调用
        WindowsEditor.CreateInstance<WindowsEditor>().Show();

        //浮动型的窗口，跟点击Building Setting出现的窗口效果一样
        WindowsEditor.CreateInstance<WindowsEditor>().ShowUtility();

        //弹出窗口时的效果
        WindowsEditor.CreateInstance<WindowsEditor>().ShowPopup();
        //此方法和下面的OnGUI配合着使用，否则会出现页面关不掉的情况,此时不会出现UI自带的关闭界面

        //使用系统提供的单例模式，在unity面板中打开窗口时，只会出现一个(推荐)
        WindowsEditor.GetWindow<WindowsEditor>().Show();
    }

    public void OnGUI()
    {
        if (GUILayout.Button("关闭"))
        {
            this.Close();
        }
    }
    
}

```

每个方法效果如下

| ![image-20231026000101143](image-20231026000101143.png) | ![image-20231026000129453](image-20231026000129453.png) | ![image-20231026000158455](image-20231026000158455.png) |
| ------------------------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------- |
| Show                                                    | ShowUtility                                             | ShowPopup                                               |

### b、界面的事件相关

自定义页面中有时会使用自定义的事件，事件通常如下：

```c#
    public int index_1 = 0;
    //刷新方法，每秒钟100次
    public void Update()
    {
        index_1++;
        Debug.Log("index_1：" + index_1);
    }

    public int index_2 = 0;
    //刷新方法，刷新周期比Update小
    public void OnInspectorUpdate()
    {
        index_2++;
        Debug.Log("index_2：" + index_2);
    }

    //视图被删除的时候触发
    public void OnDestroy()
    {
        Debug.Log("OnDestroy：触发");
    }

    //选择一个对象的时候触发
    public void OnSelectionChange()
    {
        //获取选择Hierarchy里面游戏物体的时候的名字
        for (int i = 0; i < Selection.gameObjects.Length; i++)
        {
            Debug.Log("OnSelectionChange：触发" + Selection.gameObjects[i].name);
        }

        //获取选择Project里面的文件的名字
        for (int i = 0; i < Selection.objects.Length; i++)
        {
            Debug.Log("OnSelectionChange：触发" + Selection.objects[i].name);
        }
    }

    //获得焦点的时候触发
    public void OnFocus()
    {
        Debug.Log("OnFocus：触发");
    }

    //失去焦点的时候触发
    public void OnLostFocus()
    {
        Debug.Log("OnLostFocus：触发");
    }

    //Hierarchy更改的时候触发
    public void OnHierarchyChange()
    {
        Debug.Log("OnHierarchyChange：触发");
    }

    //Project更改的时候触发
    public void OnProjectChange()
    {
        Debug.Log("OnProjectChange：触发");
    }
```

### c、文本/颜色字段

```c#

    public string 文本 = "默认文字";
    public Color 颜色 = Color.white;
    public void OnGUI()
    {
        if (GUILayout.Button("关闭"))
        {
            this.Close();
        }
        //一行文本，单行输入框
        this.文本 = EditorGUILayout.TextField(this.文本);
        //多行输入框
        this.文本 = EditorGUILayout.TextArea(this.文本);
        //密码输入框
        this.文本 = EditorGUILayout.PasswordField(this.文本);
        //颜色选择框
        this.颜色 = EditorGUILayout.ColorField(this.颜色);
    }
```

效果如下：

![image-20231026001735821](image-20231026001735821.png)

### d、标签字段

类似于[header("")]标识，在gui中显示tag。

```c#
    public string 文本 = "默认文字";
    public Color 颜色 = Color.white;
	public void OnGUI()
    {
        //标签
        EditorGUILayout.PrefixLabel(this.文本);//只写这么一行代码，窗口里面不出现任何内容
        EditorGUILayout.Space();//换行

        //标签（下拉框）
        this.文本 = EditorGUILayout.TagField(this.文本);
        EditorGUILayout.Space();

        //标签（非可选择标签提示）
        EditorGUILayout.LabelField(this.文本);
        EditorGUILayout.Space();

        //标签（可选择标签提示）
        EditorGUILayout.SelectableLabel(this.文本);
        EditorGUILayout.Space();
    }
```

效果如下：

![image-20231026002300584](image-20231026002300584.png)

### e、输入字段

```c#
   public string 文本 = "默认文字";
    public int 数字;

    public void OnGUI()
    {
        EditorGUILayout.LabelField("输入整形数字");
        this.数字 = EditorGUILayout.IntField(this.数字);
        EditorGUILayout.LabelField("输入字符");
        this.文本 = EditorGUILayout.TextField(this.文本);
    }
```

效果如下：

![image-20231026002601592](image-20231026002601592.png)

### f、滑动条

```c#
    public int 数字;
    public float 浮点数;
    public float 浮点最大值;
    public float 浮点最小值;
    public void OnGUI()
    {
        //滑动条  参数1：默认值，参数2：最小值，参数3：最大值
        this.浮点数 = EditorGUILayout.Slider(this.浮点数, 0, 100);//取值区间有小数

        this.数字 = EditorGUILayout.IntSlider(this.数字, 0, 100);//取值区间为整数

        //最小值到最大值的取值范围，为0-100
        EditorGUILayout.MinMaxSlider(ref 浮点最小值, ref 浮点最大值, 0, 100);
    }
```

效果如下：

![image-20231026003117435](image-20231026003117435.png)

### g、Vector输入字段

```c#
    public Vector3 mPos3;
    public void OnGUI()
    {
        this.mPos3 = EditorGUILayout.Vector3Field("三维坐标", this.mPos3);
        EditorGUILayout.Space();
    }
```

效果如下：

![image-20231026003443557](image-20231026003443557.png)

### h、选择

```c#
public int index;
public void OnGUI()
    {        
    	string[] strs = { "a", "b", "c", "d" ,"e"};
        int[] ints = { 1, 2, 3, 4, 5 };
        this.index = EditorGUILayout.Popup(this.index, strs);
        this.index = EditorGUILayout.IntPopup(this.index, strs,ints);
        EditorGUILayout.Space();
    }
```

![image-20231026004155844](image-20231026004155844.png)

i、对象选择

```c#
    public string mTag;
    public int mLayer;
    public Object mObject;
    public void OnGUI()
    {
        //标签，下面两种写法，哪个都可以
        //this.mTag = EditorGUILayout.TagField(this.mTag);
        this.mTag = EditorGUILayout.TagField("Tag");
        EditorGUILayout.Space();
        //层
         this.mLayer = EditorGUILayout.LayerField(this.mLayer);
         EditorGUILayout.Space();
         //对象选择，如果想要存放所有的元素，那么参数就写为Object： typeof(Object)
        this.mObject = EditorGUILayout.ObjectField("对象选择器", this.mObject, typeof(Camera), true);
   }
```



---

参考文章：

[Unity3D插件开发教程 - 挽风入我怀 - 博客园 (cnblogs.com)](https://www.cnblogs.com/yzx885059439/p/14497026.html)

[Unity 自定义Inspector面板时的数据持久化问题 - FancyBit - 博客园 (cnblogs.com)](https://www.cnblogs.com/fancybit/p/InspectorPersistantProblem.html)
