---
layout: post
title: 使用Python合并图片和Google瓦片
category : tech
description: ""
guid: e5290ceb-eba1-11e6-9a6a-acbc32c984c8
tags : [Python]
---
{% include JB/setup %}



合并PNG图片与Google瓦片，有如下几个步骤:

1. 给定PNG图片及其对应的Google坐标范围`extent`, 即`900913`或`EPSG:3857`坐标系坐标
2. 根据`extent`以及需要缩放的级别，确定合并时需要哪些瓦片
3. 对瓦片进行拼接, 生成合并后的底图
4. 根据缩放的级别调整合并后的PNG图片大小，并其放置在底图上. 需要调整PNG图片相对地图的偏移

具体的`Python`代码如下(`2.7.10`):

```python

# -*- coding:utf-8 -*-
from PIL import Image
import os
import math

tile_size = 256  # 瓦片大小256*256
tile_base_dir = "data/google/satellite" # 瓦片本地目录，也可以使用url
max_extent = [-20037508.34, -20037508.34, 20037508.345578495, 20037508.345578495] # Google坐标系的最大范围
max_resolution = 156543.0339 # Google坐标系最大分别率

def merge_tile_and_png(png_info, result_filepath):

    # 1. 根据图片坐标范围extent及缩放级别zoom，获取瓦片编号范围
    extent = png_info['extent']
    zoom = png_info['zoom']

    # 注意瓦片的编号规则
    tile_bound = get_tile_bound(extent, zoom)
    tile_dir = os.path.join(tile_base_dir, str(zoom))

    # 2. 拼接瓦片，生成底图
    color = (255, 255, 255, 0)
    width = int((tile_bound[2] - tile_bound[0] + 1)* tile_size)
    height = int((tile_bound[3] - tile_bound[1] + 1)* tile_size)

    out = Image.new('RGBA', (width, height), color)
    imx = 0;
    for x in range(tile_bound[0], tile_bound[2] + 1):
        imy = 0
        for y in range(tile_bound[1], tile_bound[3] + 1):
            tile_file = os.path.join(tile_dir, str(x), str(y) + ".png")
            if os.path.exists(tile_file):
                tile = Image.open(tile_file)
                out.paste(tile, (imx, imy))
            imy += tile_size
        imx += tile_size

    # 3. 根据zoom, 调整图片大小
    png_bound = get_png_bound(extent, tile_bound[-2:], zoom)

    png_size = (png_bound[2] - png_bound[0], png_bound[3] - png_bound[1])
    im = Image.open(png_info[‘file_path’])
    png_pic = im.resize(png_size, Image.ANTIALIAS)

    # 4. 计算PNG图片相对瓦片的偏移，并放置到底图上
    out.paste(png_pic, (png_bound[0], png_bound[1]), png_pic)

    # 5. 保存结果
    out.save(result_filepath)

def get_tile_bound(extent, zoom):
    """
    获取瓦片的编号范围
    """
	
    res = max_resolution / (2**zoom)
    min_x = int(math.floor((extent[0] - max_extent[0]) / (res * tile_size)))
    min_y = int(math.floor((max_extent[3] - extent[3]) / (res * tile_size)))
	
	max_x = int(math.ceil((extent[2] - max_extent[0]) / (res * tile_size)))
    max_y = int(math.ceil((max_extent[3] - extent[1]) / (res * tile_size)))
		 
    min_xx = min_x * tile_size
    min_yy = min_y * tile_size

    return [min_x, min_y, max_x, max_y, min_xx, min_yy]


def get_png_bound(extent, origin, zoom):
    """
    根据缩放级别， 计算PNG图片相对瓦片底图的坐标
    """
    res = max_resolution / (2 ** zoom)
    min_x = int(round((extent[0] - max_extent[0]) / res))
    min_y = int(round((max_extent[3] - extent[3]) / res))

    max_x = int(round((extent[2] - max_extent[0]) / res))
    max_y = int(round((max_extent[3] - extent[1]) / res))

    return [min_x - origin[0], min_y - origin[1],
            max_x - origin[0], max_y - origin[1]]

```

最终的效果，样例如下(PNG图为某地市建筑物分布图):

<img src="/assets/images/python/tile_combine_image.png" width="600" alt="tile-combine-image" >


*The End.*
