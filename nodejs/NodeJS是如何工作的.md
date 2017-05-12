# NodeJS 是如何工作的？

> 原文：[Eugene Obrezkov - How does NodeJS work?](https://blog.ghaiklor.com/how-nodejs-works-bfe09efc80ca)

> 译：sino

本文的代码出自于已有的 NodeJS 源码，所以在读完本文后，你将能更舒适的使用 NodeJS。
![main vision](https://cdn-images-1.medium.com/max/1000/1*CvWNEpByFHjfLjIMZfpnmg.jpeg)

## 为什么要这样做

我想你的第一个问题应该是 —— “我们为什么要这样做？”
这里我想要引用 Vyacheslav Egorov 的一句话：“The more people stop seeing JS VM as a mysterious black box that converts JavaScript source into some zeros-and-ones the better”。对于 NodeJS 同样的：“The more people stop seeing NodeJS as a mysterious black box that runs JavaScript with low-level API the better”。

## 开始吧！

时间回到 2009 年，那时的 NodeJS 刚刚起步。

我们想要在后端运行 JavaScript 并且访问底层 API，也想从 CLI 和 REPL 运行 JavaScript 程序。基本上我们想要 JavaScript 做一切事情！

那么要怎么做呢？第一件映入我脑海里的事是...

## 浏览器

浏览器可以运行 JavaScript。把浏览器集成到我们的程序里不就好了吗？

不全是！这里有很多要回答的问题。

浏览器能暴露底层 API 给 JavaScript 吗？ —— 不！

浏览器允许从别处运行 JavaScript 吗？ —— 不好说，这很复杂！

我们需要浏览器的 DOM 吗？ —— 这有点过头了！

我们需要浏览器吗？ —— 不需要！JavaScript 的执行不需要浏览器。

如果浏览器不是 JavaScript 执行所必需的话，那么要怎么执行 JavaScript 呢？

## 虚拟机（VM）

### 虚拟机执行 JavaScript

虚拟机基于高级编程语言提供了高级抽象。

虚拟机被设计为通过提供抽象和独立平台程序执行环境来执行单个计算机程序。

有很多虚拟机能够执行 JavaScript 包括谷歌的 V8，微软的 Chakra， Mozilla 的 SpiderMonkey，苹果的 JavaScriptCore 等。

我建议使用谷歌的 V8 虚拟机，为什么?因为在同类的虚拟机中它是最快的，在后端领域执行速度很重要，我想你也会认同。

那么我们瞧瞧 V8 是怎么帮助构建 NodeJS 的。

## V8 虚拟机

**V8 能被集成到任何 C++ 项目中。** 只需要将 V8 的源代码作为库包含在代码中，
你就能够使用 V8 API，它使编译运行 JavaScript 代码成为可能。

**V8 可以将 C++ 暴露给 JavaScript。** 当我们想要在 JavaScript 中使用底层 API 时，这是非常重要的。

凭以上两点就能粗略的想象出我们的构思的实现——"如何通过 JavaScript 访问底层 API".

我想概括描的述一下上面的东西，因为下一章我们会讲 C++ 代码。在我们的案例中，您可以使用虚拟机 V8 -> 将其集成到我们的 C++ 项目中 -> 借助 V8 将 C++ 暴露给 JavaScript。

但是我们要怎么写 C++ 代码才能在 JavaScript 里访问它呢？

## V8 Templates（模板）

### 通过 V8 Templates（模板）

template（模板）是 JavaScript 函数和对象的蓝图。可以使用 template 将 C++ 函数和数据结构包装到 JavaScript 对象中。

例如，chrome 使用 template 把 C++ DOM 节点包装到 JavaScript 对象中并注册函数到全局作用域。

你也能创建一组 templates 然后使用他们。因此你想要多少 templates 就有多少。

V8 有两种类型的 templates：*Function Templates*（功能模板）和 *Object Templates*（对象模板）。

**Function Templates** 是单个函数的蓝图。用过调用 template 的 *GetFunction* 方法从你希望初始化的 JavaScript 函数上下文中创建一个 JavaScript template 实例。
你也能把 C++ 回调函数与一个 function template 关联起来，当 JavaScript 实例被调用后，这个 function template 将被调用。

**Object Templates** 用来配置 function template 创建的对象作为它们的构造函数。
你可以通过 object templates 关联两种 C++ 回调函数：*accessor callback*（访问者回调）和 *interceptor callback*（拦截器回调）。
当特定的对象属性被脚本访问后 *accessor callback*（访问者回调）将被调用，当任意一对象属性被脚本访问时，*interceptor callback*（拦截器回调）将被调用。
一言蔽之，你能把 C++ 对象、结构体包装到 JavaScript 对象中。

看一个简单的例子。下面的代码将 C++ 方法 *LogCallback* 暴露到全局 JavaScript 上下文中。

```c++
// Create a template for the global object and set the built-in global functions
Local<ObjectTemplate> global = ObjectTemplate::New(isolate);
global->Set(
  String::NewFromUtf8(isolate, "log"),
  FunctionTemplate::New(isolate, LogCallback)
);

// Each processor gets its own context so different processors do not affect each other
Persistent<Context> context = Context::New(isolate, NULL, global);
```

在第二行，我们创建了 object template。然后在第三行我们创建了 function template 并将 C++ 方法 *LogCallback* 与之关联起来。
然后我们又将 function template 的实例设置到 object template 的实例上。
在第九行，我们将 object template 实例传递给新的 JavaScript 上下文，当在这个上下文中执行 JavaScript 时，
你将能够在全局作用域内调用 *LogCallback* 方法。最后，与 function template 的实例关联了起来的 C++ 方法 *LogCallback* 被触发。

如你所见，仅在 C++ 中，它类似于在 JavaScript 中的对象定义。

**到现在，我们学习了如何暴露 C++ 方法、结构体到 JavaScript中。** 我们将会学习如何在这些修改的上下文中运行 JavaScript 代码。
很简单，仅仅是编译然后运行的原则。

## V8 Compile（编译）&& Run（运行）JavaScript

如果你想在一个已创建的上下文中运行 JavaScript 代码的话，你只需要调用 V8 的两个 API —— Compile 和 Run。

让我们看一下这个例子，我们创建一个新的上下文并在其中运行 JavaScript 代码。

```cpp
// Create a new context.
Local<Context> context = Context::New(isolate);

// Enter the context for compiling and running the hello world script.
Context::Scope context_scope(context);

// Create a string containing the JavaScript source code.
Local<String> source = String::NewFromUtf8(isolate, "'Hello' + ', World!'");

// Compile the source code.
Local<Script> script = Script::Compile(source);

// Run the script to get the result.
Local<Value> result = script->Run();

// Convert the result to an UTF8 string and print it.
String::Utf8Value utf8(result);
printf("%s\n", *utf8);
```

在第二行我们创建了 JavaScript 上下文（我们可以使用上文描述的 templates 修改它）。
在第五行，我们使能这个上下文，用于编译和运行JavaScript代码。
在第八行，我们从 JavaScript 源码创建了新的字符串。它能是从硬编码中读取而来，也能是从文件或其他方式读取而来。
在第十一行，我们编译了 JavaScript 源码。在第十四行，我们运行它并预期结果。

把上面所描述的所有技术结合起来，最终，我们能构建一个简化的 NodeJS，（逃。

## C++ -> V8 Templates -> Run JavaScript -> ？

创建 VM 实例（*也就是 V8 中的 Isolate*）-> 创建 FunctionTemplate 实例，并按需分配 C++ 回调
-> 创建 ObjectTemplate 实例，并将所有已创建的 FunctionTemplate 实例分配到它上面去
-> 将全局对象的 JavaScript 上下文创建为 ObjectTemplate 实例
-> 在这个上下文里运行 JavaScript
-> 大功告成

但是在章节标题后标记的问号是什么呢？**上面的实现还有点问题。** 我们错过了一件很重要的事。

试想，你写了很多 C++ 方法（大概 10k 行），使用了 *fs, http, crypto* 等等。
我们把它们（C++ callbacks）分配到了 FunctionTemplate 的实例上，而且在 ObjectTemplate 中导入了 FunctionTemplate。
在获取到 ObjectTemplate 的 JavaScript 实例后，我们就能从 JavaScript 全局作用域访问所有 FunctionTemplate 实例。好像一切都工作正常，但是...

如果我们现在不要 *fs* 了呢？如果我们根本不需要 *crypto* 模块呢？
如果我们不要全局加载模块而是按需加载呢？为什么不把所有的 C++ 代码写进一个包含所有 C++ 回调的文件里呢？所以上面的问题指的是...

## 模块化！

所有的这些 C++ 方法应该被分割到不同的模块中，C++ 模块应该对应着 *fs, http, crypto* 这些特性。
在 JavaScript 上下文中也是相同的逻辑，在全局作用域中不应该能访问到所有的 JavaScript 模块，应该要按需加载。

基于这些最佳实践，我们需要实现我们的模块加载器，其应该能操作加载 C++ 模块和 JavaScript 模块，以便能在各自的按需加载需求。

### 从 C++ 模块加载器开始

## C++ 模块加载器

（下面会有比较多的 c++ 代码，请不要分心）

首先从一个基础的模块加载器开始。任何一个模块加载器一定有一个用来保存所有模块（或者如何获取的信息）的结构体。

我们先介绍一下用来储存模块信息的 C++ 结构体，并命名之为 *node_module*。

```cpp
struct node_module {
  int nm_version;
  unsigned int nm_flags;
  void* nm_dso_handle;
  const char* nm_filename;
  node::addon_register_func nm_register_func;
  node::addon_context_register_func nm_context_register_func;
  const char* nm_modname;
  void* nm_priv;
  struct node_module* nm_link;
};
```

我们可以把存在的模块信息存入到上面的结构体中，最后我们就有了一个包含所有可用 C++ 模块的字典（dictionary）。

我不会细致解释上面结构体里的每个字段是什么意思，你需要注意的是其中几个。
在 *nm_filename* 字段中，保存模块的文件名，我们就知道从哪加载它。
在 *nm_register_func* 和 *nm_context_register_func* 中储存当模块被加载时需要调用的函数。这些函数将负责 template 实例的初始化。
*nm_modname* 中储存模块名（非文件名）。

下一步，我们需要实现这个结构体的辅助方法。
我们可以写一个简单的方法来保存信息到我们的 *node_module* 结构体，然后在我们模块定义里使用这个方法。我们称之为 *node_module_register*。

```cpp
void node_module_register(void* m) {
  struct node_module* mp = reinterpret_cast<struct node_module*>(m);

  if (mp->nm_flags & NM_F_BUILTIN) {
    mp->nm_link = modlist_builtin;
    modlist_builtin = mp;
  } else if (!node_is_initialized) {
    // "Linked" modules are included as part of the node project.
    // Like builtins they are registered *before* node::Init runs.
    mp->nm_flags = NM_F_LINKED;
    mp->nm_link = modlist_linked;
    modlist_linked = mp;
  } else {
    modpending = mp;
  }
}
```

如你所见，我们所做的仅仅只是把模块的新信息保存到结构体 *node_module* 而已。

现在我们能通过宏来简化注册过程。申明一个用在 C++ 模块里的宏。这个宏仅仅是对 *node_module_register* 的包装。

```cpp
#define NODE_MODULE_CONTEXT_AWARE_X(modname, regfunc, priv, flags)    \
  extern "C" {                                                        \
    static node::node_module _module =                                \
    {                                                                 \
      NODE_MODULE_VERSION,                                            \
      flags,                                                          \
      NULL,                                                           \
      __FILE__,                                                       \
      NULL,                                                           \
      (node::addon_context_register_func) (regfunc),                  \
      NODE_STRINGIFY(modname),                                        \
      priv,                                                           \
      NULL                                                            \
    };                                                                \
    NODE_C_CTOR(_register_ ## modname) {                              \
      node_module_register(&_module);                                 \
    }                                                                 \
  }

#define NODE_MODULE_CONTEXT_AWARE_BUILTIN(modname, regfunc)           \
  NODE_MODULE_CONTEXT_AWARE_X(modname, regfunc, NULL, NM_F_BUILTIN)   \
```

第一个宏是对 *node_module_register* 的包装。另一个只是对前一个宏进行某些参数预定义的二次包装。
最后我们获得了一个只接收两个参数（modname 和 regfunc）的宏。当被调用时，我们就把新模块的信息保存到了 *node_module* 结构体里。
modname 和 regfunc 分别是什么意思？嗯，modname 是我们的模块名，比如 fs。
regfunc 是我们前面提到的模块的方法，这个方法负责 V8 Template 的初始化以及将它们分配给 ObjectTemplate。

如你所见，每个 C++ 模块都能用一个接受两个参数的宏申明。我们需要做的是，创建能从 *node_module* 结构体里读取信息并能调用 regfunc 方法的 C++ 方法。

那我们再写一个能从 *node_module* 结构体中通过模块名查找模块的简单方法，称之为 *get_builtin_module*。

```cpp
struct node_module* get_builtin_module(const char* name) {
  struct node_module* mp;

  for (mp = modlist_builtin; mp != nullptr; mp = mp->nm_link) {
    if (strcmp(mp->nm_modname, name) == 0)
      break;
  }

  CHECK(mp == nullptr || (mp->nm_flags & NM_F_BUILTIN) != 0);
  return (mp);
}
```

如果模块名匹配，这个函数将会从 *node_module* 结构体里返回前面申明的模块。

基于 *node_module* 结构体的信息，我们能写出一个加载 C++ 模块并且分配 V8 Template 实例到我们的 ObjectTemplate 的方法。
最后，这个 ObjectTemplate 将会作为一个 JavaScript 实例被送到 JavaScript 上下文。

```cpp
static void Binding(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);

  Local<String> module = args[0]->ToString(env->isolate());
  node::Utf8Value module_v(env->isolate(), module);

  Local<Object> cache = env->binding_cache_object();
  Local<Object> exports;

  if (cache->Has(module)) {
    exports = cache->Get(module)->ToObject(env->isolate());
    args.GetReturnValue().Set(exports);
    return;
  }

  // Append a string to process.moduleLoadList
  char buf[1024];
  snprintf(buf, sizeof(buf), "Binding %s", *module_v);

  Local<Array> modules = env->module_load_list_array();
  uint32_t l = modules->Length();
  modules->Set(l, OneByteString(env->isolate(), buf));

  node_module* mod = get_builtin_module(*module_v);
  if (mod != nullptr) {
    exports = Object::New(env->isolate());
    // Internal bindings don't have a "module" object, only exports.
    CHECK_EQ(mod->nm_register_func, nullptr);
    CHECK_NE(mod->nm_context_register_func, nullptr);
    Local<Value> unused = Undefined(env->isolate());
    mod->nm_context_register_func(exports, unused,
      env->context(), mod->nm_priv);
    cache->Set(module, exports);
  } else if (!strcmp(*module_v, "constants")) {
    exports = Object::New(env->isolate());
    DefineConstants(exports);
    cache->Set(module, exports);
  } else if (!strcmp(*module_v, "natives")) {
    exports = Object::New(env->isolate());
    DefineJavaScript(env, exports);
    cache->Set(module, exports);
  } else {
    char errmsg[1024];
    snprintf(errmsg,
             sizeof(errmsg),
             "No such module: %s",
             *module_v);
    return env->ThrowError(errmsg);
  }

  args.GetReturnValue().Set(exports);
}

env->SetMethod(process, "binding", Binding);
```

关于上面的代码的几个注释。Binding 以模块名作为参数，这个参数是你通过宏给出的。
我们通过 *get_builtin_module* 方法来获取该模块信息。如果找到了，调用该模块的初始化函数，传递一些有用的参数，比如 exports。
exports 是一个 ObjectTemplate 实例，所以我们能对它使用 V8 Template API。所有操作完成后，我们从 Binding 方法获取 exports 对象。
如你所知，ObjectTemplate 实例可以返回 JavaScript 实例，这就是 Binding 所做的。

我们要做的最后一件事使这个方法在 JavaScript 上下文中有效。
这在代码中的最后一行，通过把 Binding 方法包装到 FunctionTemplate 中，并且把它分配到全局变量 process 上来实现。

在这一步，你能通过调用 `process.binding(‘fs’)` 来获取实例，并获取其原生绑定。

这里有一个简化逻辑的内建模块的例子。

```cpp
// Include header files here
namespace node {
  // Using namespaces
  // Define useful macros
  // And a lot other stuff here in C++
  void UpdateHeapStatisticsArrayBuffer(const FunctionCallbackInfo<Value>& args) {
    // Some logic here...
  }

  void SetFlagsFromString(const FunctionCallbackInfo<Value>& args) {
    // And here...
  }

  // Our regfunc method that accepts exports ObjectTemplate as target
  void InitializeV8Bindings(Handle<Object> target, Handle<Value> unused, Handle<Context> context) {
    Environment* env = Environment::GetCurrent(context);
    env->SetMethod(target, "updateHeapStatisticsArrayBuffer", UpdateHeapStatisticsArrayBuffer);
    env->SetMethod(target, "setFlagsFromString", SetFlagsFromString);
    target->Set(
      FIXED_ONE_BYTE_STRING(env->isolate(), "heapStatisticsArrayBuffer"),
      ArrayBuffer::New(env->isolate(), env->heap_statistics_buffer(), heap_statistics_buffer_byte_length)
    );
}

NODE_MODULE_CONTEXT_AWARE_BUILTIN(v8, node::InitializeV8Bindings)
```

上面的代码将创建一个导出 JavaScript 对象的名称为“v8”的绑定，以便从 JavaScript 上下文调用 `process.binding('v8')` 获取此对象。

希望你还能跟得上。

现在我们应该使用 JavaScript 模块加载器来帮助我们完成所有像 ```require("fs")``` 这样的整齐的东西。

## JavaScript 模块加载器

得益于上面的改进，我们能调用 `process.binding()` 并且从 JavaScript 上下文访问 C++ 绑定。
但是这仍然没有解决 JavaScript 模块的问题。我们该怎么写 JavaScript 模块并按需加载呢？

首先，我们需要理解有两种模块。一种是用 C++ 写的 JavaScript 模块，简而言之，NodeJS 的内建模块，如 fs 等。我们称之为原生模块（NativeModule）。
另外一种模块在你的工作路径里，称之为模块（Module）。

我们需要能同时加载这两种模块。那就意味着我们需要知道怎么从当前的工作路径抓取原生模块和普通模块。

### 首先尝试原生模块（NativeModule）

所有的 JavaScript 原生模块都位于我们 C++ 工程的另外一个文件夹中。那就意味着所有这些 JavaScript 代码在编译时是可以访问的。
我们就能把 JavaScript 源代码包装到后面会用到的 C++ 头文件中。

在 tools 文件夹下有一个叫 js2c.py 的 python 工具，它包装 JavaScript 代码到 node_native.h 头文件中。

node_native.h 头文件可以被任意 C++ 文件包含并在 C++ 中访问 JavaScript 源代码。

现在我们可以在 C++ 上下文中使用 JavaScript 源代码 —— 让我们来试试。
我们可以实现一个简单的方法 DefineJavaScript，它从 node_natives.h 获取 JavaScript 源代码，并将它们分配给 ObjectTemplate 实例。

```cpp
void DefineJavaScript(Environment* env, Handle<Object> target) {
  HandleScope scope(env->isolate());

  for (int i = 0; natives[i].name; i++) {
    if (natives[i].source != node_native) {
      Local<String> name = String::NewFromUtf8(env->isolate(), natives[i].name);
      Handle<String> source = String::NewFromUtf8(env->isolate(), natives[i].source, String::kNormalString, natives[i].source_len);
      target->Set(name, source);
    }
  }
}
```

上面的代码中，我们遍历了每个原生 JavaScript 模块，并使用模块名为键、模块本身为值，设置到 ObjectTemplate 实例。
我们要做的最后一件事情是，以 ObjectTemplate 作为 target 调用 *DefineJavaScript*。

*Binding* 方法在这里派上了用场。如果看看我们的 *Binding* C++ 实现（C++ 模块加载器章节）的话，你将会看到我们硬编码了两个绑定：constants 和 natives。
因此，如果绑定的名称是原生的，那么使用环境和导出对象调用 *DefineJavaScript* 方法。
因此，在调用 `process.binding('natives')` 时将返回 JavaScript 原生模块。

嗯，很棒棒。还有另一个改进可以通过定义 node.gyp 中的 GYP 任务并且从中调用 js2c.py。
这将使得当 NodeJS 正在编译时，JavaScript 源也将被包装到 node_natives.h 头文件中。

到目前为止，我们有原生模块的 JavaScript 源代码可以作为 `process.binding('natives')` 使用。我们为原生模块（NativeModule）写一个简单的 JavaScript 包装。

```js
function NativeModule(id) {
  this.filename = id + '.js';
  this.id = id;
  this.exports = {};
  this.loaded = false;
}

NativeModule._source = process.binding('natives');
NativeModule._cache = {};

NativeModule.require = function(id) {
  if (id == 'native_module') {
    return NativeModule;
  }
  
  var cached = NativeModule.getCached(id);
  if (cached) {
    return cached.exports;
  }

  if (!NativeModule.exists(id)) {
    throw new Error('No such native module ' + id);
  }

  process.moduleLoadList.push('NativeModule ' + id);

  var nativeModule = new NativeModule(id);

  nativeModule.cache();
  nativeModule.compile();

  return nativeModule.exports;
};

NativeModule.getCached = function(id) {
  return NativeModule._cache[id];
};

NativeModule.exists = function(id) {
  return NativeModule._source.hasOwnProperty(id);
};

NativeModule.getSource = function(id) {
  return NativeModule._source[id];
};

NativeModule.wrap = function(script) {
  return NativeModule.wrapper[0] + script + NativeModule.wrapper[1];
};

NativeModule.wrapper = [
  '(function (exports, require, module, __filename, __dirname) { ',
  '\n});'
];

NativeModule.prototype.compile = function() {
  var source = NativeModule.getSource(this.id);
  source = NativeModule.wrap(source);
  
  var fn = runInThisContext(source, {filename: this.filename});
  fn(this.exports, NativeModule.require, this, this.filename);

  this.loaded = true;
};

NativeModule.prototype.cache = function() {
  NativeModule._cache[this.id] = this;
};
```

现在，你可以通过调用 `NativeModule.require()` 方法调用所想要加载的模块。
首先它会检查模块是否存在于 cache，如果存在 —— 直接从 cache 中获取，否则模块将被编译，缓存并作为导出对象返回。

现在再仔细看看 *cache* 和 *compile* 方法。

所有缓存都是将 NativeModule 实例设置为位于 NativeModule 中的静态对象 *_cache*。

更有趣的是 *compile* 方法。首先，我们从 *_source*（通过 `process.binding('natives')` 设置这个静态属性）获取所需模块的源代码。
然后我们使用 *wrap* 方法将它们包装在一个函数中，可以看到，result 函数接受 export，require，module， __filename 和 __dirname 参数。
之后，我们用必需的参数来调用这个函数。
结果，我们的 JavaScript 模块被包装在作为指向 NativeModule.exports 的指针的范围内，
作为指向 NativeModule.require 的指针，作为指向 NativeModule 实例本身的指针，以 __filename 作为具有当前文件名的字符串。
现在，您知道模块和加载的所有东西都来自您的 JavaScript 代码。他们只是指向 NativeModule 实例:)

### 还有普通模块（module）加载器的实现

普通模块（module）的加载器实现与原生模块（NativeModule）加载器基本相同，不同的是普通模块的源代码不是从 node_natives.h 头文件中取得，
而是从文件中读取而来。所以像包装、缓存和编译，这些都是一样的，唯一的不同是源代码取自于文件。

很好，现在我们知道如何加载原生模块或者从工作目录加载普通模块了。

最后我们用上面的所有东西再写一个每次 NodeJS 运行或者准备 NodeJS 环境时会执行的 JavaScript 模块。

## NodeJS Runtime Library ？

什么是运行时库（Runtime Library）？它是一个用来准备环境，设置全局变量  *process, console, Buffer* 等，并且在执行 NodeJS 命令行脚本时会执行的库。
它可以通过一个简单的 JavaScript 文件来实现，该文件将在所有其他 JavaScript 代码之前在 NodeJS 运行时执行。

我们开始把我们所有的原生模块代理到全局作用域并设置一些其他的全局变量。
这只是一些形如 `global.Buffer = NativeModule.require('buffer')` 或 `global.process = process` 的 JavaScript 代码。

第二步，执行你发送给 NodeJS 命令行作为参数的 main 脚本。逻辑也是非常的简单。
它只是解析了 `process.argv[1]` 这个值，并且创建了普通模块（Module）的实例作为构造函数值。
所以说，普通模块（Module）能从文件 -> 缓存读取源代码，并像原生模块（NativeModule）预编译 JavaScript 代码那样编译。

这里没有很多可以添加的，它真的简单至极，如果你想要更多详细信息，你可以瞧一瞧 node 仓库下的 src/node.js 文件。
这个文件会在 NodeJS 运行时中执行，并且使用了本文所提及的所有技术。

**这就是 NodeJS 能运行 JavaScript 代码并且访问底层 API 的原因。** 很酷，没错吧？

但是上面提及的东西还不能做任何异步的事情。像 `fs.readFile()` 这些操作现在还是完全同步的。

我们需要什么来实现异步操作呢？**事件循环**...

## 事件循环

事件循环是一个等待着事件调度或者程序信息的信息调度者。
它可以通过向某些内部或外部事件提供者（通常阻止请求直到事件到达）来请求，然后调用相关的事件处理程序（调度事件）。
事件循环可以与反应器结合使用，如果事件提供者遵循可以选择或轮询的文件接口。事件循环几乎总是与消息发起者异步运行。

当创建 V8 环境时，V8 可以接受事件循环作为一个参数。但是在设置一个事件循环到 V8 中之前，我们需要先实现它...

还好我们已经有了这个实现，它叫做 libuv。它负责像文件读写等的所有异步操作。
没有了 libuv，NodeJS 只能同步执行 JavaScript/C++ 代码。

所以我们将 libuv 源包含在 NodeJS 中，并在其中创建具有 libuv 默认事件循环的 V8 环境。这里有一个实现。

```cpp
Environment* CreateEnvironment(Isolate* isolate, uv_loop_t* loop, Handle<Context> context, int argc, const char* const* argv, int exec_argc, const char* const* exec_argv) {
  HandleScope handle_scope(isolate);

  Context::Scope context_scope(context);
  Environment* env = Environment::New(context, loop);

  isolate->SetAutorunMicrotasks(false);

  uv_check_init(env->event_loop(), env->immediate_check_handle());
  uv_unref(reinterpret_cast<uv_handle_t*>(env->immediate_check_handle()));
  uv_idle_init(env->event_loop(), env->immediate_idle_handle());
  uv_prepare_init(env->event_loop(), env->idle_prepare_handle());
  uv_check_init(env->event_loop(), env->idle_check_handle());
  uv_unref(reinterpret_cast<uv_handle_t*>(env->idle_prepare_handle()));
  uv_unref(reinterpret_cast<uv_handle_t*>(env->idle_check_handle()));

  // Register handle cleanups
  env->RegisterHandleCleanup(reinterpret_cast<uv_handle_t*>(env->immediate_check_handle()), HandleCleanup, nullptr);
  env->RegisterHandleCleanup(reinterpret_cast<uv_handle_t*>(env->immediate_idle_handle()), HandleCleanup, nullptr);
  env->RegisterHandleCleanup(reinterpret_cast<uv_handle_t*>(env->idle_prepare_handle()), HandleCleanup, nullptr);
  env->RegisterHandleCleanup(reinterpret_cast<uv_handle_t*>(env->idle_check_handle()), HandleCleanup, nullptr);

  if (v8_is_profiling) {
    StartProfilerIdleNotifier(env);
  }

  Local<FunctionTemplate> process_template = FunctionTemplate::New(isolate);
  process_template->SetClassName(FIXED_ONE_BYTE_STRING(isolate, "process"));

  Local<Object> process_object = process_template->GetFunction()->NewInstance();
  env->set_process_object(process_object);

  SetupProcessObject(env, argc, argv, exec_argc, exec_argv);
  LoadAsyncWrapperInfo(env);

  return env;
}
```

*CreateEnvironment* 方法接受 libuv 事件循环作为一个循环参数。
我们能从 V8 命名空间内调用 *Environment::New*，并且将 libuv 事件循环发送过去，然后在 V8 环境中配置它。
这就是 NodeJS 一般运行的方法。

我还想要多讲一些 libuv，告诉你它是如何工作的，但那就是之后的故事了。

## 致谢！

感谢所有阅读这篇文章的人。我希望你喜欢它，并学到新的东西。如果您发现任何问题或任何问题，请随时给予评论，我将尽快回复。
