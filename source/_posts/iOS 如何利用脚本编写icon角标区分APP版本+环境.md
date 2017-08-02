---
title: iOS 如何利用脚本编写icon角标区分APP版本+环境
date: 2017-07-01
categories:

- 项目实战

tags: 
- 分环境

---

#引子
上篇文章解决了如何快速科学的区分APP环境。紧随着又出现这个需求。
当APP存在多个环境的时候，如何快速的区分不同的版本+环境，如果能直观的通过角标来完成这样一个区分就太棒了。
<!--more-->

#本文目的
1.  解决如何通过角标区分环境 。
2. 打包build++。

#展示效果
这是在fir上面打包出来的效果。
![1507F8BB-F39F-4DB0-B081-0CB3DE7CDD75.png](http://upload-images.jianshu.io/upload_images/1743443-7999547e25d19775.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#如何实现
#####首先花几分钟安装Homebrew + Ghostscript插件，很简单
[安装教程](http://www.jianshu.com/p/ed29cd01acf6 )
 需要说的是上面这个链接简书的作者不是原创，最早是一个外国的开发者开发的，中国有人进行了翻译，然后这篇博客也被抄来抄去，但是并不影响我们使用。
#####1. 安装好了上面的两个插件之后，在项目包下面添加角标图片，名字别修改了，因为脚本里面找的路径关联图片名称

![1C423191-E05A-40EF-B61E-31EF312DBB6C.png](http://upload-images.jianshu.io/upload_images/1743443-b666563ff1e66d62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####2.  建立脚本。
注意这里脚本的顺序不要乱，就加在最后面，因为这里的脚本会按照先后顺序进行编译。如果把角标脚本放到台前面，会出问题。
![屏幕快照 2017-07-19 下午2.29.35.png](http://upload-images.jianshu.io/upload_images/1743443-e15b140fe7d8d179.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####3. 脚本编写。
(下面会给出demo，这个脚本根据不同的判断条件用来绘制角标。挺复杂的，但是并不需要弄懂才能使用，使用的时候更改环境，和对应的图片就能得到你想要的效果)
脚本最终的位置
![A0C21A2D-5F3F-4278-8784-36985850B05A.png](http://upload-images.jianshu.io/upload_images/1743443-df717f9ec7823be1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

脚本文件
```
IFS=$'\n'
//获取build,需要画到图标上面去
buildNumber=$(/usr/libexec/PlistBuddy -c "Print CFBundleVersion" "${INFOPLIST_FILE}") 
//获取版本号，需要画到图标上面去
versionNumber=$(/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" "${INFOPLIST_FILE}")
//获取bundleID, 需要进行环境的判断
bundleIdentifier=${PRODUCT_BUNDLE_IDENTIFIER}

PATH=${PATH}:/usr/local/bin

echo "this is PR Environment" //打印现在这个环境是PR环境

function generateIcon () {
    
    BASE_IMAGE_NAME=$1
    
    TARGET_PATH="${BUILT_PRODUCTS_DIR}/${UNLOCALIZED_RESOURCES_FOLDER_PATH}/${BASE_IMAGE_NAME}"
    echo "this line is target path"
    echo $TARGET_PATH
    echo $SRCROOT
    echo $(find ${SRCROOT} -name "AppIcon60x60@2x.png")
    BASE_IMAGE_PATH=$(find ${SRCROOT} -name ${BASE_IMAGE_NAME})
//进行宽高的计算
    WIDTH=$(identify -format %w ${BASE_IMAGE_PATH})
    FONT_SIZE=$(echo "$WIDTH * .15" | bc -l)
    echo "font size $FONT_SIZE"
    
//最后需要被绘制到icon上面的角标
    iconbadgeDebug="${SRCROOT}/iconbadge/debugRibbon.png"
    iconbadgeRelease="${SRCROOT}/iconbadge/prbetaRibbon.png"
    resizedIconBadge="${SRCROOT}/iconbadge/resizedRibbon.png"
    echo "iconbadge is $iconbadge"
    
//根据不同环境进行绘制
    if [ "${CONFIGURATION}" == "Debug" ]; then
    convert ${iconbadgeDebug} -resize ${WIDTH}x${WIDTH} ${resizedIconBadge}
    convert ${BASE_IMAGE_PATH} -fill white -font Times-Bold -pointsize ${FONT_SIZE} -gravity south -annotate 0 "$versionNumber.$buildNumber" - | composite ${resizedIconBadge} - ${TARGET_PATH}
    fi
    
    if [ "${CONFIGURATION}" == "Release" ]; then
    convert ${iconbadgeRelease} -resize ${WIDTH}x${WIDTH} ${resizedIconBadge}
    convert ${BASE_IMAGE_PATH} -fill white -font Times-Boldr -pointsize ${FONT_SIZE} -gravity south -annotate 0 "$versionNumber.$buildNumber" - | composite ${resizedIconBadge} - ${TARGET_PATH}
    fi
}

generateIcon "AppIcon20x20@2x.png"
generateIcon "AppIcon20x20@3x.png"
generateIcon "AppIcon29x29@2x.png"
generateIcon "AppIcon29x29@3x.png"
generateIcon "AppIcon40x40@2x.png"
generateIcon "AppIcon40x40@3x.png"
generateIcon "AppIcon60x60@2x.png"
generateIcon "AppIcon60x60@3x.png"

```
直接去demo里面复制文件，然后更改这个地方图片的路径+Debug／Realse，最终选择使用哪个角标进行绘制

![752AAF19-D60E-45CE-85D5-B5CABFF22C4D.png](http://upload-images.jianshu.io/upload_images/1743443-5c44ba41b0449a2b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意这里是会针对Target进行建立脚本
a.  APPStore不要这个脚本
b.  Debug模式下面对应Debug角标
c.  开发环境对应的角标是Bete角标
d.  Pr环境对应的是Pr角标
这是我们公司的规则，当然你也可以根据你们公司的业务进行整理

demo里面为了在debug展示不同的效果，对图片的路径进行了更改，下面就是脚本的效果

![90DFDFAC-9550-418A-BB19-36E1E6C55439.png](http://upload-images.jianshu.io/upload_images/1743443-d6d676f41f0724eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

补充：因为我们公司的项目全部在fir上面进行分发，为了记录次数以及辨认版本，我们进行了每次打包的时候build+1的脚本，也附在demo里面去了，这里就不展开了。

![87BFE8CF-5022-4F84-8FBD-1D08A7A795A1.png](http://upload-images.jianshu.io/upload_images/1743443-2bad97b8ac0f430d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果你觉得我的文章+demo对你有帮助，欢迎给github上面留下你的star，不胜感激。
[文章demo](https://github.com/knightSaber/CBTSubscriptDemo-)

如果解决文章对你有帮助，请给我的github  star 一下，不胜感激