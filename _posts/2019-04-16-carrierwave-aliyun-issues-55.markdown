---
layout: post
title:  "carrierwave-aliyun 远端文件不会随记录更新而删除的问题"
date:   2019-04-16 14:34:52 +0800
categories: rails carrierwave
---
# 问题
- rails 5.2.2.1
- carrierwave 1.3.1
- carrierwave-aliyun 0.9.0

在以上版本发现了这样一个问题：当记录删除时，阿里云 OSS 对应的文件不会被删除。

```ruby
class Package < ApplicationRecord
  mount_uploader :image, ImageUploader
end

package = Package.first

package.destroy # 记录删除, 文件删除失败

package.remove_image! # 删除失败

package.image.remove! # 删除成功
```

#  原因

正常情况下，当方法 package.remove_image! 成功调用到 package.image.remove! 时文件会删除成功。现在猜测问题就出在调用 package.remove_image! 方法后，并没有调用 package.image.remove!


一步步看源码调试，首先 remove_image! 方法是 mount_uploaders 方法动态添加的:

```ruby
def remove_#{column}!
  _mounter(:#{column}).remove
end

def _mounter(column)
  # We cannot memoize in frozen objects :(
  return Mounter.new(self, column) if frozen?
  @_mounters ||= {}
  @_mounters[column] ||= Mounter.new(self, column)
end
```
调试中可以通过 `package.image.instance_exec { _mounter(:image) }` 拿到 Mounter 的实例. 再看看 Mounter#remove! 的代码:

```ruby
def remove!
  uploaders.reject(&:blank?).each(&:remove!)
  @uploaders = []
end
```
试着运行 `package.instance_exec { _mounter(:image) }.uploaders.map(& :blank?)` 结果居然是 `[true]`. 如果 uploader.blank? = true, uploader.remove! 当然不会被调用.

而在 Uploader#blank? 的值是通过检测 file.blank? 的值:
```ruby
def blank?
  file.blank?
end
```
调试中发现，此时 file 的类则是 `CarrierWave::Storage::AliyunFile`, 翻源码这个类并没有定义 blank? 方法, 而他的父类 CarrierWave::SanitizedFile 定义了 empty? 方法

```ruby
def empty?
  @file.nil? || self.size.nil? || (self.size.zero? && ! self.exists?)
end
```

回来看 CarrierWave::Storage::AliyunFile 没有为 @file 赋值过, 也没有实现 exists？方法, 所以 file.blank? 为 nil. 也就是阿里云 OSS 文件没有被删除的原因.

# 解决

在 https://github.com/huacnlee/carrierwave-aliyun/issues/55 里提供了解决办法. 但由于这个问题存在已久, 作者未回应. 且这个 gem 依赖的 aliyun-oss-sdk 版本过于老旧, 更新的希望渺茫.

# 拓展

[Rails 对 Object 添加了 blank? 方法](https://github.com/rails/rails/blob/ec88cd626da8cefb0e11454e814f85c34b498bba/activesupport/lib/active_support/core_ext/object/blank.rb)
```ruby
def blank?
    respond_to?(:empty?) ? !!empty? : !self
end
```

所以定义了 empty? 方法也就影响了 blank? 方法的值 
