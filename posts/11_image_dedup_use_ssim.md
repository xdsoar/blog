---
title: SSIM算法图像去重
date: 2020-07-19 04:55:21
permalink: image_dedup_use_ssim
---

最近碰到一个蛮有意思的问题. 我原来在windows平台下有使用一款叫`timesnapper`的软件. 这个软件的作用是可以在间隔一段固定的时间后做一次屏幕截图, 将图片保存下来. 我用这个功能主要是用来做时间开销的记录. 此外, 之前在hackernews上看到过一个有意思的帖子是对比20年来的桌面的变化. 所以我觉得定期记录一下自己的电脑也算是蛮有意思的一件事情.

不过在我换到mac平台的时候, 没有找到一款类似的软件. 于是我写了一个很简单的定时脚本, 每隔几分钟调用一下macos自带的截屏工具保存图片. 这里的代码很简单, 大致如下

```python
#!/usr/local/env python
#coding=utf8
import datetime
import subprocess
import os


def main():
    path = '/Users/xdsoar/Documents/ScreenRecord/'
    time = str(datetime.datetime.now())
    date = time[0:10]
    month = date[0:7]
    path = path + month + '/'
    newPath = path + date + '/'
    if not os.path.exists(path):
        os.mkdir(path)
    if not os.path.exists(newPath):
        os.mkdir(newPath)
    time = time.replace(' ', '_').replace(':','.')
    file1 = newPath + time + "_1.png"
    file2 = newPath + time + "_2.png"
    file3 = newPath + time + "_3.png"
    param = "screencapture -x " + file1 + " " + file2 + " " + file3
    print("calling the cmd" + param)
    subprocess.call(['/usr/sbin/screencapture', '-x', file1, file2, file3])

if __name__ == '__main__':
    main()

```

写完以后放了一个定时任务在crontab里就算完工了. 这几年来一直运转正常. 不过有一个小毛病过去倒是一直困扰着我. pc上的timesnapper有一项功能就是检测到画面没有变化时, 就不会截屏. 我猜测原理可能是识别如键盘、鼠标的活动时间, 或者干脆就是识别画面. 这个功能可以节省非常多的冗余图片. 而我自行写的这个截图小程序自然是没有这个功能的.

于是这个周末稍微花了点时间研究了一下图像查重的方法. 很容易查到了一个叫做ssim的算法, 用于判断图片之间的相似度. 这里的相似度是基于结构相似度, 虽说可能和我的场景不是完全一致, 不过实际测试过以后发现还是比较符合预期的. 基于方便的原则, 还是用python写了一个, 基于opencv的ssim实现

```python
import os
from skimage.measure import compare_ssim
import cv2
import cProfile

def get_file_list(path):
    files = os.listdir(path)
    gray_files = []
    file_compare_result = []
    times = 0
    for file in files:
        image_file = cv2.imread(path+'/'+file)
        gray_image = cv2.cvtColor(image_file, cv2.COLOR_BGR2GRAY)
        is_dedup = False
        for i in range(0, len(gray_files)):
            gray_file = gray_files[i]
            if gray_file.shape != gray_image.shape:
                continue
            (score, _) = compare_ssim(gray_image, gray_file, full=True)
            if score > 0.99:
                gray_files.pop(i)
                gray_files.insert(0, gray_file)
                is_dedup = True
                break
        file_compare_result.append((file, is_dedup))
        if len(gray_files) > 10:
            gray_files.pop()
        gray_files.insert(0, gray_image)
        print('done once ' + str(times))
        times += 1
    return file_compare_result



if __name__ == "__main__":
    cProfile.run('result = get_file_list("/Users/xdsoar/Documents/project/image-dedup/sample")')
    print(result)
```

这段代码里用了`cProfile`, 是因为写完以后对性能有点好奇, 不知道是花在图像加载上, 还是实际的ssim算法上. 跑了一下发现基本都是花在图像加载上. 因此代码里对于图片的比较稍微做了点优化. 虽然说只比较前后两张图片的话, 时间复杂度就会很美好了. 不过实际上会遇到一种情况就是电脑锁屏以后的待机画面经常被不连续的截进来. 因此还是希望尽可能放大对比的范围. 当然如此一来, 时间复杂度就可能上升到o(n²). 对于本身就比较耗时的ssim算法而言, 时间开销就会相当庞大.

顺带一提, 我原来在mac上运行这个代码的时候, 基本上cpu都被吃满了, 因此我推测ssim算法是可以用到多线程的. 于是我尝试同样的代码和文件搬到了台式机上, 因为台式机用的是8核cpu而笔记本2核, 预期是可以有数倍的性能提升的. 结果台式机上运行反而慢过笔记本, 而且cpu并没有吃满, 只使用了10%左右. 这里我的猜测是笔记本上的opencv可能是使用了pyopencl, 用核显做了加速. 而台式机因为显卡是nvidia的, 可能并不支持加速.

于是我开始考虑如果能使用nvidia的显卡(我用的是1060)来做计算的话, 或许还能快一些. 于是开始找支持n卡加速的ssim算法库. 看了一圈发现pytorch支持n卡的cuda框架, 并且也有ssim算法的实现. 于是几经波折在pc上装上了pytorch. 代码也改成了如下版本

```python
import os
from pytorch_msssim import ssim, ms_ssim
import torch
import numpy as np
from PIL import Image
import shutil


def get_file_list(path):
    files = os.listdir(path)
    gray_files = []
    file_compare_result = []
    times = 0
    for file in files:
        if not (os.path.isfile(path+file)):
            continue
        image_file = Image.open(path + file)
        size = image_file.size
        resize_image = image_file.resize((int(size[0]/2), int(size[1]/2)),Image.ANTIALIAS)
        image_array = np.array(resize_image).astype(np.float32)
        image_torch = torch.from_numpy(image_array).unsqueeze(0).permute(0, 3, 1, 2)  # 1, C, H, W
        image_torch = image_torch.cuda()
        is_dedup = False
        for i in range(0, len(gray_files)):
            gray_file = gray_files[i]
            if gray_file.shape != image_torch.shape:
                continue
            score = ssim(image_torch, gray_file)
            if score.item() > 0.99:
                gray_files.pop(i)
                gray_files.insert(0, gray_file)
                is_dedup = True
        file_compare_result.append((file, is_dedup))
        if len(gray_files) > 100:
            gray_files.pop()
        gray_files.insert(0, image_torch)
        print('done once with ' + str(times))
        times += 1
    return file_compare_result


def compare(file1, file2):
    ima1 = to_cuda(Image.open(file1))
    ima2 = to_cuda(Image.open(file2))
    score = ssim(ima1, ima2)
    print(score.item())

def to_cuda(ima):
    image_array = np.array(ima).astype(np.float32)
    image_torch = torch.from_numpy(image_array).unsqueeze(0).permute(0, 3, 1, 2)  # 1, C, H, W
    return image_torch.cuda()


if __name__ == "__main__":
    path_arg = "C:\\Users\\qwerp\\Documents\\source\\dedup\\sample\\"
    results = get_file_list(path_arg)
    os.mkdir(path_arg + 'dedup')
    for result in results:
        if result[1]:
            shutil.move(path_arg+result[0], path_arg+'dedup/'+result[0])
    print(results)

```

结果是速度上确实有了非常感人的提升, 体感应该有5到10倍吧. 以后如果换用更好的显卡, 应该会有更好的效果. 另外在做优化的时候小开了一个脑洞, 用PIL将图像长宽都缩小到原来的一半, 结果不仅在计算速度上有非常大的提升, 而且显存能存储的图片数量也是有了一个量级的提高. 是的, 这里依旧限制了只保存100张图片进行对比, 原因就是将大量图片转换成向量以后保存在显存里还蛮大的, 未缩减大概是单张80mb. 最后的对比结果也基本符合预期, 重复的图片都找了出来. 只是这里阈值设置低一点的话(比如95), 那么一些稍微有点区别的截图也会被抓进来. 很难说哪样取舍更好. 我想最终我会使用的方案是删掉那些完全重复(重合度超过99%)的截图释放空间, 而把那些高重复度的截图单独建目录存放.

做完以后我又想到一个点子. 同样用ssim算法, 不知道能不能找出所有截图里最另类的一批? 因为图片存着以后我还是会定期回顾的. 但是大量图片的回顾确实也非常累, 如果能从大量图片中找到相对高价值的那些, 应该能节省不少精力. 关于这一点, 接下来有时间我想我会琢磨琢磨.