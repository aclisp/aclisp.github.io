# 如何在Go中快速编写单元测试

我们都喜欢单元测试，因为它们可以帮助我们保持软件的正常运行。我们都讨厌它们，因为它们看起来并不神奇-有人需要编写它们。而且在编写方面，通常需要花费大量时间来涵盖最简单的情况。

但是我找到了无痛苦的方法（好吧，减轻了痛苦）。我会像简单的插图指南一样与您分享。

## 分层次

该原理并不新，但仍然有用。如今，它带有不同的名称-六角形体系结构，洋葱体系结构，关注点分离等。主要思想是应将应用程序的不同部分（在逻辑上是不同的部分）分离为独立的部分。

这非常重要，因为您不能只测试要构建的整个应用程序。好的，从技术上讲您可以。但是，这将需要大量时间，从长远角度来看，这将是一场噩梦。

相反，使其尽可能独立。但是应用程序或至少微服务不能独立于自己！可以，我们称之为依赖注入。Go中从未像现在这样简单，因为我们拥有...

### 鸭子类型接口

这意味着，如果某种类型具有接口中描述的方法，则可以通过该接口使用它。因此，无需在项目开始时做很多文书工作并绘制具有所有可能的交互和关系的庞大UML-只需为层编写一个具有最低要求的接口并将其依赖项传递给它即可。

> 示例：您有一个业务逻辑包，必须将一些数据保存到数据库中。无需在此程序包中的某处创建或获取数据库连接-只需定义一个Repository或Storage接口（或更合适的名称），然后将要对数据库执行的所有操作放在此处即可-保存，更新，读取，删除，递增计数器等。然后，您只需放入可以执行该操作的数据库对象-它将包含确切的数据库查询语言代码和特定于数据库的逻辑。您还将在业务逻辑层之外处理数据库连接和可能的连接错误。该层将变得独立于数据库层。

### 注入非托管资源

有许多非托管资源，例如随机数生成器，时间，基于随机数的哈希等等。它们不需要外部注入，因为它们没有外部依赖项。但是它们仍然会对测试结果产生不可预测的影响。因此，与其尝试在测试中解决它们，不如使用相同的方法-将它们作为独立的依赖项注入。巴士因为不是从外面提供的，所以要从里面做！

例：

```golang
type Example struct {}

func NewExample() *Example {
    ...
    return &Example{}
}

func (e *Example) TimeToGo() string {
    now := time.Now()
    return fmt.Sprintf("its time to go! %s", now.String())
}
```

在这里，您无法预测TimeToGo测试中的方法响应-每次time.Now()都会返回一个新值。但您可以负责：

```golang
type Example struct {
    now func() time.Time
}

func NewExample() *Example {
    ...
    return &Example{time.Now}
}

func (e *Example) TimeToGo() string {
    return fmt.Sprintf("its time to go! %s", e.now().String())
}
```

将像以前一样工作，但是time.Now现在由您控制！您可以在测试中轻松模拟它：

```golang
nowMock := func() time.Time {
    t, _ := time.Parse(time.Kitchen, time.Kitchen)
    return t
}

e := Example{nowMock}

if e.TimeToGo() != "its time to go! 0000-01-01 15:04:00 +0000 UTC" {
    t.Errorf("test failed")
}
```

因此，请尝试避免对逻辑实体的任何依赖。甚至小到当前时间或随机值。可以将它们隐藏在接口或函数类型下。

## 80/20

100％的覆盖率是任何开源开发人员的梦想。在您的项目的自述文件中查看绿色徽章真是太好了！但是在生产性项目中，事情以不同的方式运作。

通常，您只是没有足够的时间或资源来进行100％的测试覆盖率。即使您愿意这样做，也要结束，在活跃的开发阶段中，您将更改逻辑多次，因此测试更改的数量将是巨大的。

但实际上，80/20原则在这里也适用：“最热门”或最重要的代码的20％覆盖率将覆盖应用程序使用率和数据流的80％。那就是说，从“最热”的代码覆盖率开始。如果您有时间和动力，您将为不太重要的部分编写测试，并慢慢增加覆盖范围。

例如，如果您要构建网络搜索服务，请首先测试搜索引擎。如果它确实按预期工作，则可以覆盖自动完成，翻译和实时预览。但是如果没有可靠的核心功能，您将失败。

## 不需要写所有的代码

现在，您已准备好应用程序中隔离得很好且非常重要的部分。您需要使测试适合该框架。幸运的是，您的应用使用了接口，而不是确切的实现。这意味着，我们可以在测试中模拟它们！

但是编写模拟游戏是一项艰巨的任务。我们不必抢机器人的工作-让他们按照自己的设计去做！

因此，我不编写测试模拟，而是生成它们。为此，我使用了Mockery。

* https://github.com/vektra/mockery

假设我们有一个数据库接口：

```golang
package repo

type Storage interface {
    GetOrder(id string) (*Order, error)
    CreateOrder(order Order) error
    DeleteOrder(id string) error
}
```

和使用它的代码：

```golang
type AccountManager struct {
    storage repo.Storage
}
```

我们可以简单地通过调用以下命令来生成模拟：

```bash
# if the interface is in the internal/repo package
mockery -name=Storage -dir=internal/repo
```

它将为您的界面生成一个完美的模拟！它将存储在与/mocks接口包目录有关的目录中（您可以通过指定-output参数进行更改）。然后，您只需要在测试中使用它：

```golang
var testOrder repo.Order
//...
storageMock := new(mocks.Storage)
storageMock.On("GetOrder", "12345").
    Return(&testOrder, nil)

// and then inject it into your code under the test
am := AccountManager{storageMock}

// execute test cases with mocked dependency ...
```

我通常使用go-generate来运行模拟生成：

```golang
//go:generate mockery -name=Storage

type Storage interface { ... }
```

然后运行go generate ./...。或者我只是将它们作为Makefile具有绝对路径的列表放入。或者，您甚至可以递归为目录中所有导出的接口生成模拟。有关更多信息，请参见库详细信息。

## 使用工具库

测试用例的编写工作艰巨而乏味。您必须准备大量数据样本，完成设置环境的无聊工作，准备HTTP请求和响应编写器，模拟服务器，存根数据等基础结构。编写每个想要的数据检查和检查确实很繁琐。相应的错误。

但是您可以使用快捷方式节省一些时间。使用快捷方式，我的意思是将测试库（如Testify）包装在干净方便的助手中来重复测试的各个部分！

* https://github.com/stretchr/testify

假设您有一个类似以下内容的常见HTTP响应：

```golang
w := httptest.NewRecorder()
//... function under test call ...

// assertion
gotBody, _ := ioutil.ReadAll(rr.Body)
gotStatus := rr.Result().StatusCode

if string(gotBody) != tt.bodyWant {
    t.Errorf("wrong response body %s want %s", string(gotBody), tt.bodyWant)
}

if gotStatus != tt.statusWant {
    t.Errorf("wrong respose status %d want %d", gotStatus, tt.statusWant)
}
```

简单吧？但是，当你需要做的是100+次就显得有点有点无聊......你不必等待“直到一个boreout！只需使用快捷方式：

```golang
import "github.com/stretchr/testify/assert"

w := httptest.NewRecorder()
//... function under test call ...

// assertion
gotBody, _ := ioutil.ReadAll(rr.Body)
gotStatus := rr.Result().StatusCode

assert.Equal(t, string(gotBody), tt.bodyWant, "wrong response body")
assert.Equal(t, gotStatus, tt.statusWant,. "wrong response status")
```

它可能以更有趣的方式使用。假设您有一个返回指针和错误的方法（Go中的常见模式）。因此，您不想在发生错误的情况下断言返回的值，因为它将导致nil指针取消引用。因此，您必须使用nil检查和错误检查等来构建一个混乱的条件……您不必：

```golang
want := "we want to get this exact value"

got, err := GiveMeMyData(input)

// if error is nill, NotNil will return true
// otherwise it will write an error message to testing.T
if assert.NotNil(t, err, "unexpected function error") {
    // here we assert the expected value
    // but only if the error is nil!
    assert.Equal(t, got, want, "unexpected function output)
}
```

您可以通过传递断言结构来缩短testing.T：

```golang
assert := assert.New(t)

/// now you don't have to pass t to each function, 500 milliseconds saved!
assert.Equal(123, 123, "they should be equal")
```

它可能看起来很简单，但是请相信我，它将在单元测试中为您节省足够的时间来喝一杯咖啡。甚至冲泡咖啡！

## 使测试有意义

我喜欢测试，但是当涉及到测试输出时，要跟踪发生的事情，哪个测试失败以及原因变得越来越困难。拥有10个或20个测试用例是可以的，但是如果您有数百个测试（或者至少有测试集，例如，如果您使用表测试），那么仅查看测试输出就很难理解出了什么问题。

为了使其更具可读性，您必须给出适当的描述。但是，您真的希望测试用例看起来像这样吗？

```golang
if got != want {
    t.Errorf("by calling a function A when there is no data in the DB table a_data and at the same time there is no incoming messages from the MQ and no idle workers in the worker pool, it returns %s but we want %s", got, want)
}
```

我希望你不要。否则请停止阅读。因为这是我最喜欢的部分。

为了获得测试输出的可读性并使其易于导航，可以使用BDD方法进行测试。我使用它的原因有很多：

它有助于将测试构造为一系列步骤或故事；
读入测试输出很不错，因为这是测试失败的完整记录。
您可以将测试用例构建成一棵树，从根公共输入到不同结果；
有可能不进行单元测试，而是为真实的业务故事编写多步骤（或不行）测试。因此，除了测试应用程序的抽象部分之外，您还可以继续进行实际工作，涵盖真实的业务流程！这也是首先关注真正必要的测试的一种好方法，因为您可以打开一个规范并为其编写测试！
因此，让我们仔细看看。对于这种测试，我使用了银杏库。它与gomega测试匹配器库配对，并带有很多测试助手和包装器（更多快捷方式！）。

* https://github.com/onsi/ginkgo

我不喜欢BDD本身，但是我真的很喜欢在测试中使用这种方法。因此，让我向您展示一些示例。

> 您需要设置一个测试套件。通常我会一次打包一次。

上面的示例可能如下所示：

```golang
import (
    . "github.com/onsi/ginkgo"
    . "github.com/onsi/gomega"
)

var _ = Describe("Function A", func() {
    When("I call the function", func() {
        Context("and there is no data in the DB table a_data", func() {
            Context("and there is no incoming messages from the MQ", func() {
                Context("and no idle workers in the worker pool", func() {
                    It("should return 123", func() {
                        got, _ := A("123")
                        Expect(got).To(Equal("123"))
                    })
                    It("should work without errors", func() {
                        _, err := A("123")
                        Expect(err).To(BeNil())
                    })
                })
            })
        })
    })
})
```

看起来更像是真实的规格，对吧？

错误输出将如下所示：

您还可以在任何步骤中添加一些数据，并且每次您的测试到达测试树的此节点时都会添加这些数据。

伪代码示例：

```
Describe some test entity
├─With data X
│ ├─And with data B
│ │ └─And with data C
│ │   ├─It should do this
│ │   └─And this
│ └─But with data D
│     └─It should do this
└─With data Y
  ├─And with data E
  │   └─It should do this
  └─And with data E*
      └─It should error
```

因此，这是一个完整的故事，您可以使用银杏来形容！例如，在这种情况下，该节点And with data B将被执行3次：

使用数据X-> 使用数据B- >使用数据C->应该执行此操作
使用数据X-> 使用数据B- >使用数据C->
使用数据X-> 使用数据B- >但是使用数据D->应该这样做
您可以用来BeforeEach()为每个步骤设置一些上下文（设置一些变量，函数，填充模拟等等）。而且，您还可以AftrrEach()在节点的末端进行清理。

图书馆有很多其他有用的功能- BeforeSuite和AfterSuite并帮助你组织你的测试更好的许多变化。

因此，这里的主要思想是将其用作对测试过程的有意义的描述，以及每个步骤的测试顺序和上下文。

# 总结

那么如何快速做呢？让我们总结一下：

1. 通过依赖注入分离各层；
1. 使用接口来做到这一点；
1. 从最热门的代码的20％开始；
1. 使用生成的模拟来模拟这些接口；
1. 使用工具库来加快简单测试的速度；
1. 使用BDD测试套件来描述和组织复杂的测试。

最后，它可能看起来像这样：

```golang
import (
    "errors"
    "temp"
    "temp/mocks"

    . "github.com/onsi/ginkgo"
    . "github.com/onsi/gomega"
    "github.com/stretchr/testify/mock"
)

var _ = Describe("Function A", func() {
    var (
        storageMock *mocks.Storage
        testOrder   temp.Order
        testErr     error
    )

    BeforeEach(func() {
        storageMock = new(mocks.Storage)
        testOrder = temp.Order("my test order")
        testErr = errors.New("test error")
    })

    When("we want to get some order", func() {
        Context("and the order exists", func() {
            BeforeEach(func() {
                storageMock.On("GetOrder", "12345").
                    Return(&testOrder, nil)
            })

            It("should return my test order without error", func() {
                sk := temp.NewStorekeeper(storageMock)
                got, err := sk.GetMyOrder("12345")

                Expect(*got).To(BeEquivalentTo("my test order"))
                Expect(err).To(BeNil())
            })
        })

        Context("and there is no order", func() {
            BeforeEach(func() {
                storageMock.On("GetOrder", mock.Anything).
                    Return(nil, testErr)
            })

            It("should return empty order with error", func() {
                sk := temp.NewStorekeeper(storageMock)
                got, err := sk.GetMyOrder("12345")

                Expect(got).To(BeNil())
                Expect(err).To(HaveOccurred())
            })
        })
    })
})
```

写起来很快。在CI日志中非常容易阅读！

