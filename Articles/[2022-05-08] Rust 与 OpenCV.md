> 原文链接: https://blog.devgenius.io/rust-and-opencv-bb0467bf35ff
>
> **翻译：[Xiaobin.Liu](https://github.com/lxbwolf)**
>
> 选题：[Xiaobin.Liu](https://github.com/lxbwolf)
>
> 本文由 [Rustt](https://Rustt.org) 翻译，[StudyRust](https://studyrust.org) 荣誉推出

# Rust 与 OpenCV

我们都知道 Rust 如此伟大的原因。然而，与 C/C++ 等老牌巨头相比，它还是有点太新太亮了，而且我们经常需要在没有适当文档的情况下使用 C++ 语言的组件（我当然更希望它们有文档，感谢社区！）。

## 背景

我们先来回答这个问题，为什么我们要关心怎么在 Rust 中运行 OpenCV 呢？为什么不直接使用 C++、Java 或 Python？

C++ 是相当古老的王者，与 Rust 或 Go 相比，编译 C++ 代码很费劲。对于那些用着 Python 长大的年轻一代来说，安装 C++ 的包似乎很中二。是的。谁愿意把时间都浪费在安装包上？尤其是现在已经有了这么多强大的包管理工具。而 Rust 的包管理工具 Cargo 就非常强大。

在 Python 中使用 OpenCV 很容易。易于安装，易于使用，有一个庞大的社区。如果你真的想做事情，Python 是个好办法。尽管 Python 语言被很多人诟病慢，但实际上（我们大多数人都知道）只有很少一部分 Python 代码是纯粹的 Python。上一句话可能会让你的有点愤怒。可怜我们这些不幸的人，在写 Python  代码时不得不承受痛苦，要么写出奇慢无比的代码，要么绑定到其他语言，让其他人免受痛苦。

> 如果你想要增加的小功能需要一个 for 循环怎么办？或者，如果你想并行地运行一些东西怎么办？Python 可以做到这一点，只是不太擅长。———— 沃·兹基硕德

## Rust OpenCV

### 安装（MacOS）

Linux 和 Windows 用户都可以按照 [这个指南](https://github.com/twistedfall/opencv-rust) 安装。对于Mac用户来说，你可以直接按照下面的超短教程来做。

让我们从安装 OpenCV 开始。不幸的是，OpenCV 不是 Rust 包。它需要你的电脑上安装有 OpenCV（C++）。然而，在 Rust 中，不需要痛苦的链接和编写 CMake 文件。Rust 中的 OpenCV（IMHO）实际上比 C++ 中简单得多，当你想引入许多依赖关系时也不会头疼（在 C++ 中你需要写一大坨 CMake 文件）。

在 macOS 上安装它是非常容易的。假设你有 brew，只要运行

```bash
brew install opencv
```

然后在 `cargo.toml` 中添加

```toml
[dependencies]
opencv = "0.63.0" # or whatever version is the latest
```

你可以参照 [opencv-rust 仓库](https://github.com/twistedfall/opencv-rust) 获得完整的安装帮助。当我安装它的时候，我遇到了一个编译的问题，但可以很容易地按照**故障排除**部分来解决。所以如果你遇到了问题，请确保在薅秃自己之前查看该部分。

*这个 OpenCV Rust 组件绑定了 C++ 的 API（这很好，因为 C 语言的 API 已经被抛弃了）。由于 Rust 可以直接与 C 语言交互，C++ 只是被包裹在一个额外的 C 语言层中，然后暴露给 Rust。*

### demo 代码

我会基于 [Makeitnow](https://www.youtube.com/channel/UC-QQTgv-P_9_8tW1m7GSUTQ) 的 [视频教程](https://www.youtube.com/watch?v=zcfixnuJFXg) 来展示我的第一个示例。对于写过 OpenCV 的开发者来说，代码相当直观。

我喜欢用 [anyhow](https://docs.rs/anyhow/latest/anyhow/) 来处理 `Result`，而不是用 `opencv::Resutl`。我们开始写代码！

```rust
use anyhow::Result; // Automatically handle the error types
use opencv::{
    prelude::*,
    videoio,
    highgui
}; // Note, the namespace of OpenCV is changed (to better or worse). It is no longer one enormous.
fn main() -> Result<()> { // Note, this is anyhow::Result
    // Open a GUI window
    highgui::named_window("window", highgui::WINDOW_FULLSCREEN)?;
    // Open the web-camera (assuming you have one)
    let mut cam = videoio::VideoCapture::new(0, videoio::CAP_ANY)?;
    let mut frame = Mat::default(); // This array will store the web-cam data
    // Read the camera
    // and display in the window
    loop {
        cam.read(&mut frame)?;
        highgui::imshow("window", &frame)?;
        let key = highgui::wait_key(1)?;
        if key == 113 { // quit with q
            break;
        }
    }
    Ok(())
}
```

非常棒！我们打开了一个摄像头并把结果帧保存到了变量 `frame` 中。代码非常简洁，是自解释的。如果你看不懂，请去看视频！

如果用 Python 代码，可以这么写

```python
import cv2
vid = cv2.VideoCapture(0)
while True:
    ret, frame = vid.read()
    cv2.imshow('window', frame)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break
vid.release()
cv2.destroyAllWindows()
```

### 进一步熟悉 OpenCV-Rust

我们接下来做

- 从文件中读取图片
- 使用 SIFT 和 ORB 检测关键元素
- 使用不同颜色画出关键元素
- 画出一个矩形
- 把图片数据转成 `ndarray`
- 把 `ndarray` 转成 `image::RgbImage` 以验证我们上面的处理没有问题
- 保存图片

我先把所有的代码都放出来，之后再一步步分解

```rust
use anyhow::anyhow;
use anyhow::Result;
use image::RgbImage;
use ndarray::{Array1, ArrayView1, ArrayView3};
use opencv::{self as cv, prelude::*};
fn main() -> Result<()> {
    // Read image
    let img = cv::imgcodecs::imread("./assets/demo_img.png", cv::imgcodecs::IMREAD_COLOR)?;
    // Use Orb
    let mut orb = <dyn cv::features2d::ORB>::create(
        500,
        1.2,
        8,
        31,
        0,
        2,
        cv::features2d::ORB_ScoreType::HARRIS_SCORE,
        31,
        20,
    )?;
    let mut orb_keypoints = cv::core::Vector::default();
    let mut orb_desc = cv::core::Mat::default();
    let mut dst_img = cv::core::Mat::default();
    let mask = cv::core::Mat::default();
    orb.detect_and_compute(&img, &mask, &mut orb_keypoints, &mut orb_desc, false)?;
    cv::features2d::draw_keypoints(
        &img,
        &orb_keypoints,
        &mut dst_img,
        cv::core::VecN([0., 255., 0., 255.]),
        cv::features2d::DrawMatchesFlags::DEFAULT,
    )?;
    cv::imgproc::rectangle(
        &mut dst_img,
        cv::core::Rect::from_points(cv::core::Point::new(0, 0), cv::core::Point::new(50, 50)),
        cv::core::VecN([255., 0., 0., 0.]),
        -1,
        cv::imgproc::LINE_8,
        0,
    )?;
    // Use SIFT
    let mut sift = cv::features2d::SIFT::create(0, 3, 0.04, 10., 1.6)?;
    let mut sift_keypoints = cv::core::Vector::default();
    let mut sift_desc = cv::core::Mat::default();
    sift.detect_and_compute(&img, &mask, &mut sift_keypoints, &mut sift_desc, false)?;
    cv::features2d::draw_keypoints(
        &dst_img.clone(),
        &sift_keypoints,
        &mut dst_img,
        cv::core::VecN([0., 0., 255., 255.]),
        cv::features2d::DrawMatchesFlags::DEFAULT,
    )?;
    // Write image using OpenCV
    cv::imgcodecs::imwrite("./tmp.png", &dst_img, &cv::core::Vector::default())?;
    // Convert :: cv::core::Mat -> ndarray::ArrayView3
    let a = dst_img.try_as_array()?;
    // Convert :: ndarray::ArrayView3 -> RgbImage
    // Note, this require copy as RgbImage will own the data
    let test_image = array_to_image(a);
    // Note, the colors will be swapped (BGR <-> RGB)
  	// Will need to swap the channels before
    // converting to RGBImage
    // But since this is only a demo that
    // it indeed works to convert cv::core::Mat -> ndarray::ArrayView3
    // I'll let it be
    test_image.save("out.png")?;
Ok(())
}
trait AsArray {
    fn try_as_array(&self) -> Result<ArrayView3<u8>>;
}
impl AsArray for cv::core::Mat {
    fn try_as_array(&self) -> Result<ArrayView3<u8>> {
        if !self.is_continuous() {
            return Err(anyhow!("Mat is not continuous"));
        }
        let bytes = self.data_bytes()?;
        let size = self.size()?;
        let a = ArrayView3::from_shape((size.height as usize, size.width as usize, 3), bytes)?;
        Ok(a)
    }
}
// From Stack Overflow: https://stackoverflow.com/questions/56762026/how-to-save-ndarray-in-rust-as-image
fn array_to_image(arr: ArrayView3<u8>) -> RgbImage {
    assert!(arr.is_standard_layout());
let (height, width, _) = arr.dim();
    let raw = arr.to_slice().expect("Failed to extract slice from array");
RgbImage::from_raw(width as u32, height as u32, raw.to_vec())
        .expect("container should have the right size for the image dimensions")
}
```

#### 读取图片

你不知道那么多类型都是干什么的？请参阅下面的部分。它是一个非常详细的教程。

读取图片非常简单。你可能想检查下图片是否加载成功。如果没找到图片，OpenCV 不会抛出异常。不要被 Rust 中的 `Result` 迷惑了，它并不会检查图片是否加载成功了。

Rust 代码

```rust
// Read image
let img = opencv::imgcodecs::imread("./assets/demo_img.png", cv::imgcodecs::IMREAD_COLOR)?;
```

C++ 代码

```cpp
cv::Mat I = cv::imread("./assets/demo_img.png", 0);
```

Python 代码

```python
img: np.ndarray = cv2.imread("./assets/demo_img.png)
```

#### 检测关键元素并渲染

ORB 和 SIFT 的代码很相似，所以这里我只介绍 ORB。

首先创建 `detector`

Rust 代码

```rust
let mut orb = <dyn cv::features2d::ORB>::create(
        500,
        1.2,
        8,
        31,
        0,
        2,
        cv::features2d::ORB_ScoreType::HARRIS_SCORE,
        31,
        20,
    )?;
```

C++ 代码

```cpp
cv::Ptr<cv::ORB> orbPtr = cv::ORB::create();
```

这部分 Rust 与 C++ 的代码很像，只是用了一些不同的命名空间和参数。

*你可以在 Rust 文档中找到所有的默认变量，如果用的是 VSCode，将鼠标悬停在 `create` 函数处，就可以看到相关的文档了。*



![在 VSCode中可以看到 C++ 默认的参数](https://gitee.com/rustcn/rustt-assets/raw/main/20220508 Rust and OpenCV/01.png)

#### 计算关键元素

对比 Rust 与 C++

Rust

```rust
let mut orb_keypoints = cv::core::Vector::default();
let mut orb_desc = cv::core::Mat::default();
orb.detect_and_compute(&img, &mask, &mut orb_keypoints, &mut orb_desc, false)?;
```

C++ 

```cpp
std::vector<cv::KeyPoint> keypoints;
orbPtr->detect(image, keypoints);
cv::Mat desc;
orbPtr->compute( image, keypoints, desc );
```

还是没有太大的不同。初学者可能不知道怎么初始化关键元素和描述符。但是看了上面的代码后，就会发现很简单，而且 Rust 代码并不会比 C++ 代码复杂。

#### 画出关键元素

Rust

```rust
let mut dst_img = cv::core::Mat::default();
cv::features2d::draw_keypoints(
        &img,
        &orb_keypoints,
        &mut dst_img,
        cv::core::VecN([0., 255., 0., 255.]),
        cv::features2d::DrawMatchesFlags::DEFAULT,
 )?;
```

C++

```cpp
cv::Mat dst_img;
cv::drawKeypoints(image, keypoints, dst_img);
```

弄清楚 Rust 中的类型是有点困难的。使用类型推理和一点点猜测，就有可能弄明白！

#### 探索 —— 画出矩形（更有指导意义的步骤）

由于 Rust 中的 OpenCV 几乎没有文档，所以需要我们去探索。我展示 C++ 代码是有原因的。我们可以看到，利用强大的 IDE（感谢 LSP），大部分的 Rust 代码可以（在某种程度上）从 C++ 代码中推断出来。此外，Rust 有一个很好的文档系统。让我们利用 opencv-rust 的文档和 C++ 的命名来弄清楚如何画出矩形。

首先，要搞清楚我们的目标是什么：

- 画出矩形（在 C++ 中，它的类型是 `cv::rectangle`）

然后，我们去查找 [opencv-rust 文档](https://docs.rs/opencv/latest/opencv/index.html)

![02](https://gitee.com/rustcn/rustt-assets/raw/main/20220508 Rust and OpenCV/02.png)

![03](https://gitee.com/rustcn/rustt-assets/raw/main/20220508 Rust and OpenCV/03.png)

现在，我们需要做的就是了解这些类型（说起来容易做起来难）。我们将会用到 IDE （我用的是 VSCode），如 LSP、Nvim 以及 Emacs。第一个参数很明显是图片，类型是 `cv::core::Mat`。明显吗？（并没有很明显。）然而我们可以推断出来。`Mat` 是保存图片数据的默认类型，因此它就应该是 `Mat` 类型。并且 `Mat` 实现了 `ToInputOutputArray` 特性。`Rect` 又是什么类型？

![04](https://gitee.com/rustcn/rustt-assets/raw/main/20220508 Rust and OpenCV/04.png)

貌似我们可以在 `core` 包中找到它。但是我们怎么构建一个 `Rect` 呢？

LSP 可以帮我们自动生成构造函数。现在我们通过一些推断加运气很快地搞清楚了（C++ 中就没有这么简单了）。如果你用 C++，那么就可能需要“面向 Google 编程”了。

![05](https://gitee.com/rustcn/rustt-assets/raw/main/20220508 Rust and OpenCV/05.png)

我们看下 `from_points` 构造函数是什么样的

![06](https://gitee.com/rustcn/rustt-assets/raw/main/20220508 Rust and OpenCV/06.png)

我们需要两个 `Point` 类型的 `point`，但是 `Point` 类型又是什么？

![07](https://gitee.com/rustcn/rustt-assets/raw/main/20220508 Rust and OpenCV/07.png)

`core` 中貌似有个 `Point`。我们还是借助 LSP 吧！

![08](https://gitee.com/rustcn/rustt-assets/raw/main/20220508 Rust and OpenCV/08.png)

貌似是 `new`，但是 `from_vec2` 看起来也很像！可能两个都可以。我们先来试下 `new`（通常情况下，先去找 `new` 或 `from*`）

![09](https://gitee.com/rustcn/rustt-assets/raw/main/20220508 Rust and OpenCV/09.png)

我们传入两个整型参数，看看会有结果（应该会编译成功）！现在我们搞清楚了第二个参数！

```rust
cv::core::Rect::from_points(
  cv::core::Point::new(0, 0),
  cv::core::Point::new(50, 50
)
```

（其他的参数照葫芦画瓢）这个过程可能漫长又乏味，但是当你搞清楚这些类型后，其他的事情都是水到渠成了，你也不会觉得 Rust 不如 C++ 了。

#### 转成 `ndarray`

*ndarray 似乎是 Rust 上最好用的矩阵库（对于那些想封装 Python 的 NumPy 包少男少女们来说，ndarray 是个好东西）。稍后还有一篇关于 Python 的 NumPy 在 Rust 中的实现的文章!*

这就有点棘手了。现在我们需要将一个 C++ 类型转换为 Rust 类型。我们只知道我们面对的是一个矩阵类型，而它们通常是以行为主实现的。OpenCV 的 Mat 和 ndarray Array（View）都是行为主的。而数据（通常）是以序列的形式存储在底层的缓冲区中。为了确保能正确使用，我们来检查下 Mat 是否是连续的。

```rust
if !mat.is_continuous() {
    return Err(anyhow!("Mat is not continuous :("));
}
```

我们可以利用这些知识来快速（无复制，零成本）将 `cv::Mat` 转化为 `ndarray::Array`。然而，需要注意的是，数组指向的是存储在 `Mat` 中的数据。所以，当 `Mat` 被删除时，`Array` 也必须被删除。否则，我们（很可能）会指向已释放的内存，这可不是什么好事。但是，Rust 似乎为我们处理了这个问题！

首先，我们要提取 `Mat` 的数据。由于图像是以 8 位无符号整数（u8）的形式存储的，我们可以直接读取数据而不需要进行类型转换（没错！）。

```rust
let data_bytes: &[u8] = mat.data_bytes()?; // <-- This is the image data in sequence! Note it is pointing to the data in mat
```

然后我们需要计算出数据占用的内存大小。我们是以一个长序列的形式而不是以我们期望的长宽组合的形式来获取数据的。

```rust
let size = mat.size()?;
let h:i32 = size.height;
let w:i32 = size.width;
```

利用这些信息我们可以构建一个 `ArrayView3<u8>`。

```rust
let a = ArrayView3::from_shape((h as usize, w as usize, 3), data_bytes)?; // The 3 is because we have bgr. For gray image this will be 1
```

很好！我们知道了高性能从 `Mat` 转成 `ArrayView3<u8>` 的方法。**请注意，这种方式只适用于内存连续的数组。**

作为一个 Rust 的特性，它应该是这样的：

```rust
trait AsArray {
    fn try_as_array(&self) -> Result<ArrayView3<u8>>;
}
impl AsArray for cv::core::Mat {
    fn try_as_array(&self) -> Result<ArrayView3<u8>> {
        if !self.is_continuous() {
            return Err(anyhow!("Mat is not continuous"));
        }
        let bytes = self.data_bytes()?;
        let size = self.size()?;
        let a = ArrayView3::from_shape((size.height as usize, size.width as usize, 3), bytes)?;
        Ok(a)
    }
}
```

现在我们可以通过下面的调用来快速实现从 `Mat` 转成 `Array`：

```rust
let array: ArrayView<u8> = mat.try_as_array()?;
```

### 总结：

在 Rust 中使用 OpenCV 是肯定可行的。它需要更深入的知识，以便能够在不同的类型之间进行转换，并需要有足够的意志力来弄清各个类型的关系。但它是可行的。:))

由于 Rust 的包管理器 Cargo 非常强大，它鼓励使用其他人的包（比如 [cv-convert](https://crates.io/crates/cv-convert) 可以在许多流行的包之间转换图片类型）。我希望在未来能看到更多很酷的包。一些 Rust 的 GPU 包开始崭露头角，谁知道呢，也许将来可以直接将 Rust 编译到 SPIR-V 上，以实现真正的快速计算！这将是一个多么美好的未来啊。

如果你一路走到这里，感谢你花时间阅读，希望你在路上学到了一些东西。
