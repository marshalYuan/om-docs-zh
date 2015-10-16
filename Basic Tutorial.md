这个教程假设你已经熟悉Clojure或者ClojureScript了。如果你还不熟悉他们，我们建议你先阅读下[Light Table编辑器下的ClojureScript教程](http://github.com/swannodette/lt-cljs-tutorial)。当然，首先快速浏览下[Om概念预览](https://github.com/omcljs/om/wiki/Conceptual-overview),对于一个Om新手来说也是极好的。

我们将使用[Figwheel](https://github.com/bhauman/lein-figwheel)去构建一个交互式的ClojureScript和Om的开发环境。Figwheel有以下两个作用：

* 在你保存源码文件时，自动编译并重新加载你的ClojureScript和CSS文件，无需手动刷新 （类似于livereload）
* 构建一个能连接到你的浏览器的REPL，让你可以在命令行内运行一些表达式甚至操纵你正在运行的应用

安装下Clojure的项目构建工具[Leiningen](http://leiningen.org/)，我们就可以开始了。接下来，在终端内切换到你想要创建demo项目的目录，并且运行如下命令：

```lein new figwheel om-tut -- --om```

执行完后，将会创建一个包含了Om和Figwheel的叫着`om-tut`的项目。`cd`到这个目录，然后运行：

```lein figwheel```

这个命令将会自动构建一个ClojureScript项目。第一次构建大约需要几秒钟的时间。一旦构建成功，将会自动在你的默认浏览器中打开[http://localhost:3449/](http://localhost:3449/)（在这里，我们建议使用google的chrome，因为他很好地支持source maps）。你应该可以看到一个包含`Hello World!`的H1标签在这个页面。



在你喜欢的编辑器下打开`src/om_tut/core.cljs`(我用的是包含cider插件的Emacs，你也可以用Light Table)。然后改变`app-state`中`:text`
的值为其它任何字符串，保存文件。刷新你的浏览器，你将看到新的内容。为什么我们要手动刷新浏览器呢？因为这里使用了`defonce`去定义`app-state`。这意味着他阻止reload当重新设置state的时候。

这也太无聊了吧？让我们来尝试下代码热更新。调整你的应用程序窗口位置，以至于你能同时看到chrome窗口和你的源码编辑器窗口。

现在，试着把`dom/h1`变成`dom/p`,保存文件，注意你的浏览器窗口。他应该已经自动重新加载了，打开控制台，你应该也会看到打印的Figwheel日志。 这个代码改变并没有触及`app-state`。为了编辑`app-state`,你需要刷新浏览器去重置它，或者在与浏览器相连的REPL中直接操作它。在你运行`lein figwheel`的终端内，你可以看到一个REPL。如果没有，尝试刷新下你的浏览器。你可以像下面这样简单地尝试下REPL：

```cljs.user> (js/alert "Am I connected?")```

你应该可以在你的浏览器中看到这个alert。当你点击“确定”之后，这个表单时会在REPL中返回一个`nil`。这一步中，如果你看到一些异常或者错误信息，可以在运行之前排除下Figwheel的浏览器REPL的故障。如果一切安好，我们可以将命名空间从`cljs.user`切换到`om-tut.core`(我们所有代码存在的空间)，然后改变state：

```
cljs.user=> (in-ns 'om-tut.core)
nil
om-tut.core=> (swap! app-state assoc :text "Do it live!")
```

##OM 基础

在OM中，应用程序的state被保存在一种叫做`atom`的引用类型中(这里需要clojure或cljs的一点基础)。 如果你通过`swap!`或者`reset!`方法去改变这个atom的value，这通常会触发一次所有包含该state的Om根节点的re-render（虚拟dom的重绘，我们稍后会解释它）。你可以简单地将atom视为一个你客户端应用程序的数据库。在atom中的一切数据应该是一个关联的数据结构--一个ClojureScript的map或者有索引的如vector一样的连续数据结构(但不是一个set)。这意味着你不能将一个list或者一个惰性序列放入应用程序的state。在应用程序状态更新索引序列时，特别容易忘记这一点。

### om.core/root

`om.core/root`(这里的别名为om/root)，可以建立一个Om的渲染loop在一个指定的DOM元素上。`om.root`表达式在此时的用法看起来如下所示：

```
(om/root
  (fn [data owner]
    (reify
      om/IRender
      (render [_]
        (dom/p nil (:text data)))))
  app-state
  {:target (. js/document (getElementById "app"))})
```
`om.core/root`是一个幂等得方法，这意味着，无论对其求值运算多少次都是安全的（这也是为什么我们不需要`defonce`）。它接收三个参数。第一个参数是一个接收这个应用程序的state数据和返回的React Component(这里叫做owner)作为参数的方法，这个方法必须返回一个Om Component也就是实现了`om/IRender`接口的model，像`om.core/component`宏生产的一样。第二个参数是一个应用程序的state atom。第三个参数是一个map，它必须包含一个key为`:target`的DOM节点。它也可以处理一些其他的有意思的参数，我们稍后再介绍。

一个应用程序可以同时有多个root。编辑`resources/public/index.html`,把`<div id="app"><h2>Figwheel..</h2><p>Checkout..</p></div>`换为：

```
 <div id="app0"></div>

 <div id="app1"></div>
```

你需要刷新浏览器在改变`resources/public/index.html`之后，因为Figwheel不会处理HTML文件的重加载。然后编辑`src/om_tut/core.cljs`把om/root表达式改变为:

```
(om/root
  (fn [data owner]
    (om/component (dom/h2 nil (:text data))))
  app-state
  {:target (. js/document (getElementById "app0"))})
```

刷新你的浏览器。你现在只能在页面上看到一个h2标签。拷贝粘贴om/root表达式，然后编辑第二个表达式为如下样子：

```
(om/root
  (fn [data owner]
    (om/component (dom/h2 nil (:text data))))
  app-state
  {:target (. js/document (getElementById "app1"))}) ;; <-- "app0" to "app1"
```

你应该可以看到第二个h2标签神奇地在你保存文件后出现了。

在REPL中运行：

```om-tut.core=> (swap! app-state assoc :text "Multiple roots!")```

你应该可以看到两个h2标签都在更新。Om对多个root可以完美的支持，并且可以在同一个[requestAnimationFrame](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestAnimationFrame)中同步渲染。

好了，这个示例已经结束了。在继续探索Om之前，让我们先把`resources/public/index.html`中的`<div id="app1"></div>`和第二个`om/root`表达式。然后保存刷新浏览器，就可以继续下面的内容了。

##Rendering a list of things

改变`app-state`为如下表达式，然后刷新浏览器：

```(defonce app-state (atom {:list ["Lion" "Zebra" "Buffalo" "Antelope"]}))```

如果我们不是使用`defonce`，出于Figwheel的代码重加载机制，我们每次改变代码时，state都会重启。如果我们想在当前的`app-state`中尝试新的代码，那将是不可能的。

现在我们改变下`om/root`表达式然后保存。不要抱怨刷新，[John McCarthy](http://library.stanford.edu/collections/john-mccarthy-papers-0)(Lisp 之父)会生气的。

```
(om/root
  (fn [data owner]
    (om/component
      (apply dom/ul nil
        (map (fn [text] (dom/li nil text)) (:list data)))))
  app-state
  {:target (. js/document (getElementById "app0"))})
```
是不是可以看到一个动物的清单了。

你可能对`dom/ul`和`dom/li`的第一个参数是`nil`感到好奇。这个参数其实就是你设置给这个DOM的属性。熟悉React的同学，很容易联想到`React.createElement(type, props, children)`和`React.DOM.ul(props, children)`里的props参数。现在我们再做一些小改动：

```
(om/root
  (fn [data owner]
    (om/component
      (apply dom/ul #js {:className "animals"}
        (map (fn [text] (dom/li nil text)) (:list data)))))
  app-state
  {:target (. js/document (getElementById "app0"))})
```
如果你在chrome浏览器里右键审查元素，DOM中的这个ul标签应该已经有了一个“animals”的class属性。

需要注意的是`#js {...}`和`#js [...]` 标示一个js的数据字面量。Clojurescript支持通过`#js`支持Javascript的数据字面量。

其中，`#js {...}`是js中的Object：

```
#js {:foo "bar"}  ;; is equivalent to
#js {"foo" "bar"}
```
`#js [...]`是js中的Array:

```
#js [1 2 3]
```
为什么会这样呢？因为Clojurescript中seq都是持久化的数据集，而有时我们只需要js中普通的数据结构，这时我们就可以使用#js。
当然#js是一个浅层的限定符，如下：

```#js {:foo [1 2 3]} ;; 一个包含clojure持久化向量的JS object```

##Life without a templating language
在Om框架下，你可以使用Clojurescript的强大功能去构建用户界面。同时，Om也向你敞开了使用可选的语法去构建DOM的大门，如果这些语法也是你的菜的话。

让我们修改下我们的代码，来构建一个有斑马纹的list。首先，我们在`om/root`前添加一个辅助函数stripe：

```
(defn stripe [text bgc]
  (let [st #js {:backgroundColor bgc}]
    (dom/li #js {:style st} text)))
```
然后，修改下原来的root并保存：

```
(om/root
  (fn [data owner]
    (om/component
      (apply dom/ul #js {:className "animals"}
        (map stripe (:list data) (cycle ["#ff0" "#fff"])))))
  app-state
  {:target (. js/document (getElementById "app0"))})
```
正如所见，Clojurescript提供了大量让大多数模板语言相形见绌的强大编程工具。

##Your first Om component
现在，把`<div id="app0"></div>`改为`<div id="contacts"></div>`，然后移除`src/om_tut/core.cljs`中的`stripe`函数并保存。

然后把om/root修改为：

```
(om/root contacts-view app-state
  {:target (. js/document (getElementById "contacts"))})
```
先别急着保存，我们还没定义`contacts-view`，我们先来编辑下app-state：

```
(defonce app-state
  (atom
    {:contacts
     [{:first "Ben" :last "Bitdiddle" :email "benb@mit.edu"}
      {:first "Alyssa" :middle-initial "P" :last "Hacker" :email "aphacker@mit.edu"}
      {:first "Eva" :middle "Lu" :last "Ator" :email "eval@mit.edu"}
      {:first "Louis" :last "Reasoner" :email "prolog@mit.edu"}
      {:first "Cy" :middle-initial "D" :last "Effect" :email "bugs@mit.edu"}
      {:first "Lem" :middle-initial "E" :last "Tweakit" :email "morebugs@mit.edu"}]}))
```
然后定义下`contacts-view`:

```
(defn contacts-view [data owner]
  (reify
    om/IRender
    (render [this]
      (dom/div nil
        (dom/h2 nil "Contact list")
        (apply dom/ul nil
          (om/build-all contact-view (:contacts data)))))))
```
在构建Om组件时，我们可以用`om.core/build`构建单一组件，用`om.core/build-all`方法构建多个组件。在这个例子里，我们希望展示一个联系人列表，所以我们使用一个联系人的向量作为[cursor](https://github.com/omcljs/om/wiki/Cursors)。`contacts-view`返回一个包含`h2`和`ul`标签的`div`。我们想渲染几个li元素，因此对om/ul使用了apply。如果你想渲染单一组件，cursor可能不是一个向量，你只需要使用`om.core/build`方法构建组件。

好了，万事俱备只欠东风，我们来写一个`contact-view`，并把他放在`contacts-view`之后。

```
(defn contact-view [contact owner]
  (reify
    om/IRender
    (render [this]
      (dom/li nil (display-name contact)))))
```

好了，照旧，保存文件刷新浏览器。你会看到没有任何变化，而且在浏览器控制台上有一些错误错误信息。Figwheel拒绝重载存在警告信息的代码。控制台的信息如下：

```
WARNING: Use of undeclared Var om-tut.core/contact-view at line 24 src/om_tut/core.cljs
WARNING: Use of undeclared Var om-tut.core/display-name at line 30 src/om_tut/core.cljs
```
它提醒我们`contact-view`还没有定义，因为他在`contacts-view`之后定义的。因此，我们调整下`contact-view`的位置，然后保存看下警告信息是否还在。现在，还有一个提醒我们缺省`display-name`的警告。

那我们就来定义一个`display-name`，并放在`contact-view`之前。

```
(defn display-name [{:keys [first last] :as contact}]
  (str last ", " first (middle-name contact)))
```
这里，我们浅尝了一下clojure的map解构。非常方便是吧！ 最后，我们再来写一个`middle-name`的helper，它应该放在`display-name`之前。

```
(defn middle-name [{:keys [middle middle-initial]}]
  (cond
    middle (str " " middle)
    middle-initial (str " " middle-initial ".")))
```
再次使用了map解构。

保存之后，你应该就可以看到一个完整的联系人列表。

##Enhancing your first Om component

现在，我们把contacts删除。修改下`contact-view`:

```
(defn contact-view [contact owner]
  (reify
    om/IRender
    (render [this]
      (dom/li nil
        (dom/span nil (display-name contact))
        (dom/button nil "Delete")))))
```

保存一下，你就可以看到一个删除按钮，点击一下，如你所想的那样，什么也不会发生。

联系人列表不需要删除自身的能力，然而他们需要有能与有这个能力的组件实体通信的能力。

##Intercomponent communication

为了实现组件间通信的功能，我们将引入core.async的channel机制。现在，我们吧命名空间修改为:

```
(ns ^:figwheel-always om-tut.core
  (:require-macros [cljs.core.async.macros :refer [go]])
  (:require [om.core :as om :include-macros true]
            [om.dom :as dom :include-macros true]
            [cljs.core.async :refer [put! chan <!]]))
```

照旧，保存文件刷新浏览器。（注意：为了这一步正常，这里已经添加了依赖`[org.clojure/core.async "x.x.x"]`在文件`project.clj`中。如果你要在未来的项目中手动添加这个依赖的时候，你必须要先重启或停止`lein figwheel`进程）。修改下`contact-view`:

```
(defn contact-view [contact owner]
  (reify
    om/IRenderState
    (render-state [this {:keys [delete]}]
      (dom/li nil
        (dom/span nil (display-name contact))
        (dom/button #js {:onClick (fn [e] (put! delete @contact))} "Delete")
```

注意，我们把`om/IRender`修改成了`om/IRenderState`。这是因为我们把将要从中接收delete信号的channel作为这个组件的state。另外，我们为这个删除按钮添加了一个`onClick`的可以把当前contact写入channel的响应方法。稍后我们将看到这确实是一个bug。

对`contacts-view`做一个大改动，别担心，我们一点一点地了解这些改动的意义。

```
(defn contacts-view [data owner]
  (reify
    om/IInitState
    (init-state [_]
      {:delete (chan)})
    om/IWillMount
    (will-mount [_]
      (let [delete (om/get-state owner :delete)]
        (go (loop []
          (let [contact (<! delete)]
            (om/transact! data :contacts
              (fn [xs] (vec (remove #(= contact %) xs))))
            (recur))))))
    om/IRenderState
    (render-state [this {:keys [delete]}]
      (dom/div nil
        (dom/h2 nil "Contact list")
        (apply dom/ul nil
          (om/build-all contact-view (:contacts data)
            {:init-state {:delete delete}}))))))
```
** 提示： **注意这里我们使用了`vec`方法去把`remove`（一个惰性序列）的结果重新转变为vector向量，服从我们之前提到的state只能由像maps和vectors这样的联合数据结构组成的铁律。

首先，我们通过实现`om/IInitState`接口来初始化state。很简单，我们仅仅就分配了一个core.async的channel。这里有一个常见错误，我们不能在`reify`周围用一个`let`绑定来完成这个分配工作。注意这个`contacts-view`方法将可能被调用很多很多次。同样地，我们也把`om/IRender`修改成了`om/IRenderState`，因为我们希望通过他来获取这个delete channel并传递下去。

然后，我们实现了`om/IWillMount`接口，在实现方法里，我们构建了一个go循环去监听从子contact-view传来的事件。一旦我们得到了一个删除事件，我们就用`om.core/transact!`方法从应用的state里移除相关的contact。好了，现在你就可以从联系人列表里轻松地删除一些人了。

##Adding Contacts

既然我们已经有了删除功能，那么现在是时候修改下我们的程序让它可以添加新的联系人了吧！首先修改下命名空间，然后重新运行下`lein figwheel`并刷新下浏览器：

```
(ns ^:figwheel-always om-tut.core
  (:require-macros [cljs.core.async.macros :refer [go]])
  (:require [om.core :as om :include-macros true]
            [om.dom :as dom :include-macros true]
            [cljs.core.async :refer [put! chan <!]]
            [clojure.data :as data]
            [clojure.string :as string]))
```

接着我们添加一个叫`parse-contact`的方法并保存。

```
(defn parse-contact [contact-str]
  (let [[first middle last :as parts] (string/split contact-str #"\s+")
        [first last middle] (if (nil? last) [first middle] [first last middle])
        middle (when middle (string/replace middle "." ""))
        c (if middle (count middle) 0)]
    (when (>= (count parts) 2)
      (cond-> {:first first :last last}
        (== c 1) (assoc :middle-initial middle)
        (>= c 2) (assoc :middle middle)))))
```

当然，这里有很多种方式去实现一个这种功能的`parse-contact`,而且这肯定也不是最后的实现，但是它展示了一些clojure编程的惯用风格。如果你并不是一个Clojure或ClojureScript的老手，在执行这个方法之前，花一些时间来读懂上面这个实现是非常值得的。

在REPL中，尝试给这个方法一些输入，看看结果！
```
(in-ns 'om-tut.core)
(parse-contact "Gerald J. Sussman")
```

如果Figwheel抛出一个带有`clojure is not defined`或`string is not defined`的ReferenceError，关闭这个REPL，然后执行：

`lein clean`

...重启Figwheel然后刷新浏览器，这个问题就会解决。

一旦，你看到这个方法已经能正常工作，我们就可以着手写一个`add-contact`。看起来就像这样：

```
(defn add-contact [data owner]
  (let [new-contact (-> (om/get-node owner "new-contact")
                        .-value
                        parse-contact)]
    (when new-contact
      (om/transact! data :contacts #(conj % new-contact)))))
```
这一步，我们需要使用`om.core/get-node`方法来获取文本输入框中的值。我们看看接下来我们的contacts-view：

```
(defn contacts-view [data owner]
  (reify
    om/IInitState
    (init-state [_]
      {:delete (chan)})
    om/IWillMount
    (will-mount [_]
      (let [delete (om/get-state owner :delete)]
        (go (loop []
              (let [contact (<! delete)]
                (om/transact! data :contacts
                  (fn [xs] (vec (remove #(= contact %) xs))))
                (recur))))))
    om/IRenderState
    (render-state [this state]
      (dom/div nil
        (dom/h2 nil "Contact list")
        (apply dom/ul nil
          (om/build-all contact-view (:contacts data)
            {:init-state state}))
        (dom/div nil
          (dom/input #js {:type "text" :ref "new-contact"})
          (dom/button #js {:onClick #(add-contact data owner)} "Add contact"))))))
```
注意这个input的特殊字段`:ref`,这是一个源自React的特性，用来标示一小部分你需要直接访问的DOM节点。

保存文件。现在，你应该已经可以添加任意长度的联系人了，只要你至少提供姓氏和名字。

这里信息量比较大。这里我想设置一个挑战任务：当一个新的联系人已经被添加之后，清空下输入框。这个任务比想象的要稍难一些，所以不要轻易放弃。如果你在思考这个问题上花了15，20分钟后就轻松进入下一章，我们将介绍怎么实现这个需求。

##Dealing with text input fields

受React的声明模型，在Om中处理输入都有点更具挑战性，也更灵活。“最简单”的清空文本输入框的方法大概就是把add-contact修改为：

```
(defn add-contact [data owner]
  (let [input (om/get-node owner "new-contact")
        new-contact (-> input .-value parse-contact)]
    (when new-contact
      (om/transact! data :contacts #(conj % new-contact))
      (set! (.-value input) ""))))
```
这种方式运行正常，但是任何时候你发现自己严重依赖refs(引用)，那么此时很可能需要倒退回去，并考虑下是否做相同的工作可以用更加声明式的方式来完成。

因为contacts-view"拥有"这个文本输入框，因此我们可以考虑把它的值作为contacts-view的state的一部分。来吧，我们再来改一下contacts-view：

```
(defn contacts-view [data owner]
  (reify
    om/IInitState
    (init-state [_]
      {:delete (chan)
       :text ""})
    om/IWillMount
    (will-mount [_]
      (let [delete (om/get-state owner :delete)]
        (go (loop []
              (let [contact (<! delete)]
                (om/transact! data :contacts
                  (fn [xs] (vec (remove #(= contact %) xs))))
                (recur))))))
    om/IRenderState
    (render-state [this state]
      (dom/div nil
        (dom/h2 nil "Contact list")
        (apply dom/ul nil
          (om/build-all contact-view (:contacts data)
            {:init-state state}))
        (dom/div nil
          (dom/input #js {:type "text" :ref "new-contact" :value (:text state)})
          (dom/button #js {:onClick #(add-contact data owner)} "Add contact"))))))
```
保存后，试下在输入框里输入点什么。

呀！怎么什么也输入不了。让我们花一点时间来考虑下到底怎么了。

我们刚才往contacts-view的state里加了一些新东西。造成的状况是，不管用户输入什么，我们都会把输入框的值设为state中`:text`的值。所以，我们需要让它的值和输入框的值保持一致。让我们再次修改下contacts-view，加入监听输入框值的变化事件的代码：

```
(defn contacts-view [data owner]
  (reify
    om/IInitState
    (init-state [_]
      {:delete (chan)
       :text ""})
    om/IWillMount
    (will-mount [_]
      (let [delete (om/get-state owner :delete)]
        (go (loop []
              (let [contact (<! delete)]
                (om/transact! data :contacts
                  (fn [xs] (vec (remove #(= contact %) xs))))
                (recur))))))
    om/IRenderState
    (render-state [this state]
      (dom/div nil
        (dom/h2 nil "Contact list")
        (apply dom/ul nil
          (om/build-all contact-view (:contacts data)
            {:init-state state}))
        (dom/div nil
          (dom/input
            #js {:type "text" :ref "new-contact" :value (:text state)
                 :onChange #(handle-change % owner state)})
          (dom/button #js {:onClick #(add-contact data owner)} "Add contact"))))))
```
在保存之前，我们再在contacts-view之前添加一个handle-change函数：

```
(defn handle-change [e owner {:keys [text]}]
  (om/set-state! owner :text (.. e -target -value)))
```
好了，现在可以保存这些改动了。你应该又可以愉快地使用刚才的输入框了。

最后，让我们再加一小段代码去清空输入框。如你所见，这种做法很像我们开始那个“简易”尝试的做法，除了我们不再直接操作这个引用(ref)而是改变这个应用的state：

```
(defn add-contact [data owner]
  (let [new-contact (-> (om/get-node owner "new-contact")
                        .-value
                        parse-contact)]
    (when new-contact
      (om/transact! data :contacts #(conj % new-contact))
      (om/set-state! owner :text ""))))
```
我知道你想说这看起来像是一个事倍功半的做法，除了刚才我们看到的我们可以真正地细粒度控制用户输入的内容。例如，如果要求名字中不能含有数字，我们可以通过修改下`handle-change`来阻止这种行为：

```
(defn handle-change [e owner {:keys [text]}]
  (let [value (.. e -target -value)]
    (if-not (re-find #"[0-9]" value)
      (om/set-state! owner :text value)
      (om/set-state! owner :text text))))
```
现在保存一下。你可以试着输入一个名字，一旦你尝试输入一个数字，那么什么改变也不会发生。是不是非常的顺畅！

** 注意： **如果你对React很熟悉的话，你会发现这里的实现跟React相比稍显笨重，在这里我们必须去设置文本框的state即使我们并不想它改变的时候。其实这是Om针对React的内部机制做出的每次渲染都放在requestAnimationFrame中的优化带来的副作用。

## Higher Order Components

大部分强大的组件都强大在它们的子组件可以被参数化。为了专注于这一概念，让我们先撇开已有的用户输入等并发症。作为一个挑战，你应该在把这些基础文件重新添加之前，先把这一部分阅读完。

好的，我们开始重新开始吧。你的`resources/public/index.html`应该被修改为：

```
<!DOCTYPE html>
<html>
  <head>
    <link href="css/style.css" rel="stylesheet" type="text/css">
  </head>
  <body>
    <div id="registry"></div>
    <script src="js/compiled/om_tut.js" type="text/javascript"></script>
  </body>
</html>
```
你的源文件：

```
ns ^:figwheel-always om-tut.core
  (:require [om.core :as om :include-macros true]
            [om.dom :as dom :include-macros true]
            [clojure.string :as string]))

(enable-console-print!)

(def app-state
  (atom
    {:people
     [{:type :student :first "Ben" :last "Bitdiddle" :email "benb@mit.edu"}
      {:type :student :first "Alyssa" :middle-initial "P" :last "Hacker"
       :email "aphacker@mit.edu"}
      {:type :professor :first "Gerald" :middle "Jay" :last "Sussman"
       :email "metacirc@mit.edu" :classes [:6001 :6946]}
      {:type :student :first "Eva" :middle "Lu" :last "Ator" :email "eval@mit.edu"}
      {:type :student :first "Louis" :last "Reasoner" :email "prolog@mit.edu"}
      {:type :professor :first "Hal" :last "Abelson" :email "evalapply@mit.edu"
       :classes [:6001]}]
     :classes
     {:6001 "The Structure and Interpretation of Computer Programs"
      :6946 "The Structure and Interpretation of Classical Mechanics"
      :1806 "Linear Algebra"}}))

(defn middle-name [{:keys [middle middle-initial]}]
  (cond
    middle (str " " middle)
    middle-initial (str " " middle-initial ".")))

(defn display-name [{:keys [first last] :as contact}]
  (str last ", " first (middle-name contact)))

(defn registry-view [data owner]
  (reify
    om/IRenderState
    (render-state [_ state]
      (dom/div nil
        (dom/h2 nil "Registry")))))

(om/root registry-view app-state
  {:target (. js/document (getElementById "registry"))})

(defn on-js-reload []
  ;; optionally touch your app-state to force rerendering depending on
  ;; your application
  ;; (swap! app-state update-in [:__figwheel_counter] inc)
)
```
现在，我们希望registry-view根据不同类型的人渲染不同的视图，而不是通过硬编码的方式去实现。在此，我们将看到在ClojureScript的多重方法(multimethod)是如何的大放异彩。

在display-name之后，我们添加如下代码：

```
(defmulti entry-view (fn [person _] (:type person)))

(defmethod entry-view :student
  [person owner] (student-view person owner))

(defmethod entry-view :professor
  [person owner] (professor-view person owner))
```

不要急着保存代码，因为我们还没有定义student-view和professor-view方法。花一点时间，让这个想法沉淀一下，我们添加了一个间接层，使entry-view可以委派到其他我们通过`defmethod`提供的view上去。

让我们再entry-view之前写几个底层落地的view吧：

```
(defn student-view [student owner]
  (reify
    om/IRender
    (render [_]
      (dom/li nil (display-name student)))))

(defn professor-view [professor owner]
  (reify
    om/IRender
    (render [_]
      (dom/li nil
        (dom/div nil (display-name professor))
        (dom/label nil "Classes")
        (apply dom/ul nil
          (map #(dom/li nil %) (:classes professor)))))))
```

最后，我们对registry-view稍作修改，使其更加简洁，条理更清晰。

```
defn registry-view [data owner]
  (reify
    om/IRender
    (render [_]
      (dom/div #js {:id "registry"}
        (dom/h2 nil "Registry")
        (apply dom/ul nil
          (om/build-all entry-view (people data)))))))
```

好了，万事俱备，还差一个people方法。我们希望渲染professor的时候带着他们准确地课程编号列表。所以，在registry-view前添加：

```
(defn people [data]
  (->> data
    :people
    (mapv (fn [x]
            (if (:classes x)
              (update-in x [:classes]
                (fn [cs] (mapv (:classes data) cs)))
               x)))))
```

把这些都保存一下，你应该已经看到这个结果了。

希望通过Om提供的视图合成和扩展性会使你眼前一亮。如果你愿意花几个小时专注于此的话，你将会收获更多此类亮点。

不然的话，在接下来的章节中，我们将向您展示我们如何可以很容易地修改每个教授的班级名单，并在屏幕上的多个位置的变化可见。

##Interactivity & Higher Order Components

再次修改下`resources/public/index.html`:

```
<!DOCTYPE html>
<html>
  <head>
    <link href="css/style.css" rel="stylesheet" type="text/css">
  </head>
  <body>
    <div id="registry"></div>
    <div id="classes"></div>
    <script src="js/compiled/om_tut.js" type="text/javascript"></script>
  </body>
</html
```
还有`resources/public/css/style.css`:

```
ul li input {
    width: 400px;
}
ul li button {
    margin-left: 10px;
}
```

然后在registry-view后添加一个`class-view`:

```
(defn classes-view [data owner]
  (reify
    om/IRender
    (render [_]
      (dom/div #js {:id "classes"}
        (dom/h2 nil "Classes")
        (apply dom/ul nil
          (map #(dom/li nil %) (vals (:classes data))))))))
```

然后我们添加到一个新的`om/root`表达式并用上前面定义的view：

```
(om/root classes-view app-state
  {:target (. js/document (getElementById "classes"))})
```

保存这些文件，你应该会看到一些新的UI。

现在，让我们的课程名可以在行内编辑的。为此，我们希望构建一个可以在任何我们希望的地方插入的可重用的组件。这将是我们迄今所看到的最复杂的组件。

当用户编辑一个课程名时，我们应该隐藏原来的文本然后展现一个输入框。因此，我们需要一点helper方法去完成这部分工作。在app-state后添加`display`：

```
(defn display [show]
  (if show
    #js {}
    #js {:display "none"}))
```
当用户编辑一个课程名时，每按下一次按键，我们都希望去更新当前应用的state：

```
(defn handle-change [e text owner]
  (om/transact! text (fn [_] (.. e -target -value))))
```
另外，当输入框失去焦点的时候，我们希望能推出编辑模式：

```
(defn commit-change [text owner]
  (om/set-state! owner :editing false))
```
这就是我们待会儿写的`editable`组件所需的几个helper方法。`editable`接受一个javascript的字符串并且展现它，同时使它是可编辑的。为了实现这个需求，javascript的string需要去支持Om的cursor接口。庆幸的是，我们不需要亲自去实现这个接口，而是通过让javascript的字符串去实现`ICloneable`来使Om来替我们完成这部分困难的工作（注意以下的用法仅用于示范，不建议在实际应用中运用，你可以参阅一下[中级教程](http://github.com/swannodette/om/wiki/Intermediate-Tutorial)中的一个不需要扩展javascript的原生字符串到`ICloneable*`的更优秀的方法）。

** 一定要保证`extend-type`句式（译者注：lisp中form的一种译法）放在`om/root`句式之前。 **把它们放在靠近文件顶部的位置不失为一个好办法。

```
(extend-type string
  ICloneable
  (-clone [s] (js/String. s)))
```

悲剧的是，光这样还是不够的，因为javascript的字符串对象和javascript的原始字符串并不等价：

```
(extend-type js/String
  ICloneable
  (-clone [s] (js/String. s))
  om/IValue
  (-value [s] (str s)))
```
我们稍后会解释一下`IValue`。此时，会触发一个Clojurescript编译器的一个警告。正常来说，我们不能对这些警告视而不见，但是在这里，为了能够保持`editable`足够的简洁，我们不得不搞出这些幺蛾子。因为Figwheel在默认配置下，并不会对存在警告的代码进行重载，所以我们需要在`project.clj`中添加一段这样的配置：

```
:figwheel { :load-warninged-code true  ;; <- Add this
            :on-jsload "om-tut-v1.core/on-js-reload" }
```
运行一下`lein clean`(以防万一)，然后重启`lein figwheel`。

下面就是这个editable组件；看起来有点复杂，但花一点时间来阅读它，你会发现其实很简单。

```
(defn editable [text owner]
  (reify
    om/IInitState
    (init-state [_]
      {:editing false})
    om/IRenderState
    (render-state [_ {:keys [editing]}]
      (dom/li nil
        (dom/span #js {:style (display (not editing))} (om/value text))
        (dom/input
          #js {:style (display editing)
               :value (om/value text)
               :onChange #(handle-change % text owner)
               :onKeyDown #(when (= (.-key %) "Enter")
                              (commit-change text owner))
               :onBlur (fn [e] (commit-change text owner))})
        (dom/button
          #js {:style (display (not editing))
               :onClick #(om/set-state! owner :editing true)}
          "Edit")))))
```
我们这里必须使用`om.core/value`,因为React不知道怎么处理Javascript的字符串对象。这也是我们前面为什么要实现`IValue`的原因。

同样因为课程名可能是一个字符串对象而不是一个原始字符串，我们需要更改下`professor-view`：

```
(defn professor-view [professor owner]
  (reify
    om/IRender
    (render [_]
      (dom/li nil
        (dom/div nil (display-name professor))
        (dom/label nil "Classes")
        (apply dom/ul nil
          (map #(dom/li nil (om/value %)) (:classes professor)))))))
```

来，我们试下在`classes-view`中使用`editable`:

```
(defn classes-view [data owner]
  (reify
    om/IRender
    (render [_]
      (dom/div #js {:id "classes"}
        (dom/h2 nil "Classes")
        (apply dom/ul nil
          (om/build-all editable (vals (:classes data))))))))
```
好了，保存一下。你现在可以在`classes-view`中编辑课程标题。值得关注的是，在`registry-view`中的课程标题与此保持着完美的同步。

来一个挑战，在`professor-view`中使用一下`editable`来渲染课程标题而不是简单的字符串。这仅仅需要对代码进行一行的改变。如果你完成了这个任务，想想看在“刚好工作”的`registry-view`内部是如何编辑课程标题的(译者注：这句翻译有偏差)。这是Om的cursors提供的模块性完成的。





