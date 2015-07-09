title: cocos2dx中的坑
categories: 引擎
tags: [cocos2dx]
---
####1. tmx地块图片文件支持相对路径，但是却没有对..,.此类相对路径符号过滤处理，导致后面android中打开资源失败。####
>注意 AAssetManager_open中打开的文件必须是相对于assets的绝对路径，不能包含..，.符号，AAssetManager_open内部实现会在此路径前附加assets/，并且使用字符串完全匹配，即使是/改为\也跪。

代码：CTMXXMLParser.cpp中解析tmx文件中<image source="图片路径"
``` 
else if (elementName == "image")
{
    TMXTilesetInfo* tileset = tmxMapInfo->getTilesets().back();

    // build full path
    std::string imagename = attributeDict["source"].asString();

    if (_TMXFileName.find_last_of("/") != string::npos)
    {
        string dir = _TMXFileName.substr(0, _TMXFileName.find_last_of("/") + 1);
        tileset->_sourceImage = dir + imagename;
    }
    else
    {
        tileset->_sourceImage = _resources + (_resources.size() ? "/" : "") + imagename;
    }
}
```
这里tmx文件包含路径，就要在内部图片文件前附加此路径。
*处理办法是直接将这个判断除掉，并且内部图片文件直接填写相对于assets的路径，例如assets/img/map/1.png，就直接填img/map/1.png，这个文件名直接传递给AAssetManager_open。*
***
