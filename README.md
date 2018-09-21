# CompressImageDemo

概要:

图片的两种压缩方法
1.1 压缩图片质量
1.2 压缩图片尺寸
压缩图片使图片文件小于指定大小
2.1 压缩图片质量
2.2 压缩图片尺寸
2.3 两种图片压缩方法结合
文章更新记录

2017-12-17日

1. 修改实现方法(改为UIImage的分类实现)更新demo工程
一、两种图片压缩方法

两种压缩图片的方法：压缩图片质量(Quality)，压缩图片尺寸(Size)。

1.1 压缩图片质量

NSData *data = UIImageJPEGRepresentation(image, compression);
UIImage *resultImage = [UIImage imageWithData:data];
通过UIImage和NSData的相互转化，减小 JPEG 图片的质量来压缩图片。UIImageJPEGRepresentation::第二个参数compression 取值 0.0~1.0，值越小表示图片质量越低，图片文件自然越小。

1.2 压缩图片尺寸

UIGraphicsBeginImageContext(size);
[image drawInRect:CGRectMake(0, 0, size.width, size.height)];
resultImage = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
给定所需的图片尺寸 size，resultImage 即为原图 image 绘制为 size 大小的图片。

二、 压缩图片使图片文件小于指定大小

如果对图片清晰度要求不高，要求图片的上传、下载速度快的话，上传图片前需要压缩图片。压缩到什么程度要看具体情况，但一般会设定一个图片文件最大值，例如100 KB。可以用上诉两种方法来压缩图片。假设图片转化来的 NSData对象为data，通过data.length即可得到图片的字节大小。

2.1 压缩图片质量

比较容易想到的方法是，通过循环来逐渐减小图片质量，直到图片稍小于指定大小(maxLength)。

UIImage+WLCompress.m

- (NSData *)compressQualityWithMaxLength:(NSInteger)maxLength {
CGFloat compression = 1;
NSData *data = UIImageJPEGRepresentation(self, compression);
while (data.length > maxLength && compression > 0) {
compression -= 0.02;
data = UIImageJPEGRepresentation(self, compression); // When compression less than a value, this code dose not work
}
return data;
}
这样循环次数多，效率低，耗时长。

可以通过二分法来优化。

UIImage+WLCompress.m

- (NSData *)compressQualityWithMaxLength:(NSInteger)maxLength {
CGFloat compression = 1;
NSData *data = UIImageJPEGRepresentation(self, compression);
if (data.length < maxLength) return data;
CGFloat max = 1;
CGFloat min = 0;
for (int i = 0; i < 6; ++i) {
compression = (max + min) / 2;
data = UIImageJPEGRepresentation(self, compression);
if (data.length < maxLength * 0.9) {
min = compression;
} else if (data.length > maxLength) {
max = compression;
} else {
break;
}
}
return data;
}
当图片大小小于 maxLength，大于 maxLength * 0.9 时，不再继续压缩。最多压缩 6 次，1/(2^6) = 0.015625 < 0.02，也能达到每次循环 compression减小 0.02 的效果。这样的压缩次数比循环减小 compression少，耗时短。需要注意的是，当图片质量低于一定程度时，继续压缩没有效果。也就是说，compression继续减小，data 也不再继续减小。

压缩图片质量的优点在于，尽可能保留图片清晰度，图片不会明显模糊；
缺点在于，不能保证图片压缩后小于指定大小。
2.2 压缩图片尺寸

与之前类似，比较容易想到的方法是，通过循环逐渐减小图片尺寸，直到图片稍小于指定大小(maxLength)。具体代码省略。同样的问题是循环次数多，效率低，耗时长。可以用二分法来提高效率，具体代码省略。这里介绍另外一种方法，比二分法更好，压缩次数少，而且可以使图片压缩后刚好小于指定大小(不只是 < maxLength， > maxLength * 0.9)。

objectiveC:

-(NSData *)compressBySizeWithMaxLength:(NSUInteger)maxLength{
UIImage *resultImage = self;
NSData *data = UIImageJPEGRepresentation(resultImage, 1);
NSUInteger lastDataLength = 0;
while (data.length > maxLength && data.length != lastDataLength) {
lastDataLength = data.length;
CGFloat ratio = (CGFloat)maxLength / data.length;
CGSize size = CGSizeMake((NSUInteger)(resultImage.size.width * sqrtf(ratio)),
(NSUInteger)(resultImage.size.height * sqrtf(ratio))); // Use NSUInteger to prevent white blank
UIGraphicsBeginImageContext(size);
// Use image to draw (drawInRect:), image is larger but more compression time
// Use result image to draw, image is smaller but less compression time
[resultImage drawInRect:CGRectMake(0, 0, size.width, size.height)];
resultImage = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
data = UIImageJPEGRepresentation(resultImage, 1);
}
return data;
}
[resultImage drawInRect:CGRectMake(0, 0, size.width, size.height)];
是用新图 resultImage 绘制，也可以用原图 image 来绘制。
用原图绘制，压缩后图片更接近指定大小，但是压缩次数较多，耗时较长。一张大小为 6064 KB 的图片，压缩图片尺寸，原图绘制与新图绘制结果如下



两种绘制方法压缩后大小很接近，与指定大小也很接近，但原图绘制压缩次数可达到新图绘制压缩次数的两倍。建议使用新图绘制，减少压缩次数。压缩后图片明显比压缩质量模糊。
需要注意的是绘制尺寸的代码CGSize size = CGSizeMake((NSUInteger)(resultImage.size.width * sqrtf(ratio)), (NSUInteger)(resultImage.size.height * sqrtf(ratio)));，每次绘制的尺寸 size，要把宽 width 和 高 height 转换为整数，防止绘制出的图片有白边。

压缩图片尺寸可以使图片小于指定大小，但会使图片明显模糊(比压缩图片质量模糊)。

2.3 两种图片压缩方法结合

如果要保证图片清晰度，建议选择压缩图片质量。如果要使图片一定小于指定大小，压缩图片尺寸可以满足。对于后一种需求，还可以先压缩图片质量，如果已经小于指定大小，就可得到清晰的图片，否则再压缩图片尺寸。

UIImage+WLCompress.m

-(NSData *)compressWithMaxLength:(NSUInteger)maxLength{
// Compress by quality
CGFloat compression = 1;
NSData *data = UIImageJPEGRepresentation(self, compression);
//NSLog(@"Before compressing quality, image size = %ld KB",data.length/1024);
if (data.length < maxLength) return data;

CGFloat max = 1;
CGFloat min = 0;
for (int i = 0; i < 6; ++i) {
compression = (max + min) / 2;
data = UIImageJPEGRepresentation(self, compression);
//NSLog(@"Compression = %.1f", compression);
//NSLog(@"In compressing quality loop, image size = %ld KB", data.length / 1024);
if (data.length < maxLength * 0.9) {
min = compression;
} else if (data.length > maxLength) {
max = compression;
} else {
break;
}
}
//NSLog(@"After compressing quality, image size = %ld KB", data.length / 1024);
if (data.length < maxLength) return data;
UIImage *resultImage = [UIImage imageWithData:data];
// Compress by size
NSUInteger lastDataLength = 0;
while (data.length > maxLength && data.length != lastDataLength) {
lastDataLength = data.length;
CGFloat ratio = (CGFloat)maxLength / data.length;
//NSLog(@"Ratio = %.1f", ratio);
CGSize size = CGSizeMake((NSUInteger)(resultImage.size.width * sqrtf(ratio)),
(NSUInteger)(resultImage.size.height * sqrtf(ratio))); // Use NSUInteger to prevent white blank
UIGraphicsBeginImageContext(size);
[resultImage drawInRect:CGRectMake(0, 0, size.width, size.height)];
resultImage = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
data = UIImageJPEGRepresentation(resultImage, compression);
//NSLog(@"In compressing size loop, image size = %ld KB", data.length / 1024);
}
//NSLog(@"After compressing size loop, image size = %ld KB", data.length / 1024);
return data;
}

