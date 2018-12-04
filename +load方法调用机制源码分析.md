###+loa的方法调用机制源码分析

+load在整个文件被加载到运行时，在 main 函数调用之前被 ObjC 运行时调用的钩子方法。
####load方法的调用栈：
- 新镜像加载到runtime时的回调：
	 _objc_init在初始化的时候，会注册一个回调，当有新的镜像加载到runtime的时候，该回调会在load_images方法中进行调用，调用load_images方法时会把最新镜像的信息列表infoList作为入参传入：


	const char *
	load_images(enum dyld_image_states state, uint32_t infoCount,
            const struct dyld_image_info infoList[]) {
    bool found;

    found = false;
    for (uint32_t i = 0; i < infoCount; i++) {
        if (hasLoadMethods((const headerType *)infoList[i].imageLoadAddress)) {
            found = true;
            break;
        }
    }
    if (!found) return nil;

    recursive_mutex_locker_t lock(loadMethodLock);

    {
        rwlock_writer_t lock2(runtimeLock);
        found = load_images_nolock(state, infoCount, infoList);
    }

    if (found) {
        call_load_methods();
    }

    return nil;
    }

- 什么是镜像？
	
    镜像并不是一个OC文件，可以理解成是一个target的编译产物
    在load_images方法中打印传入的infoList镜像信息，可以看到具体的内容:
    
    ```
    (const dyld_image_info) $55 = {
  imageLoadAddress = 0x0000000100000000
  imageFilePath = 0x00007fff5fbff8f0 "/Users/apple/Library/Developer/Xcode/DerivedData/objc-dibgivkseuawonexgbqssmdszazo/Build/Products/Debug/debug-objc"
  imageFileModDate = 0
}
```
	其中imageFilePath 都是对应的二进制文件的地址，到其目录下查看，其实是个可执行文件，和xcode打印的东西一样

- +load方法的准备阶段
	load_images方法中，扫描镜像信息时，关于+load的符号判断：
    
	```
	for (uint32_t i = 0; i < infoCount; i++) {
   	 if (hasLoadMethods((const headerType *)infoList[i].imageLoadAddress)) 	{
        	found = true;
        	break;
   	 }
	}

	```
    其中判断条件会调用load_images_nolock 来查找 load 方法，具体实现如下：
    ```
/**
 * 判断一个镜像文件是发实现了+load方法。
 * @params state 镜像文件对应的状态，枚举值
 * @params infoCount 信息数量
 * @params dyld_image_info 具体信息数组
 */
bool load_images_nolock(enum dyld_image_states state,uint32_t infoCount,
                        const struct dyld_image_info infoList[]) {
    bool found = NO;
    uint32_t i;
    
    i = infoCount;
    while (i--) {
        //imageLoadAddress镜像文件对应的内存地址
        const headerType *mhdr = (headerType*)infoList[i].imageLoadAddress;
        //判断该类是否实现了+load方法
        if (!hasLoadMethods(mhdr)) continue;
        //对+load方法进行调用前的准备
        prepare_load_methods(mhdr);
        found = YES;
    }
    
    return found;
}

	```
    其中prepare_load_methods方法负责对+load方法做调用前的最后准备，具体实现如下：
    ```
	/**
 	* +load 方法调用前的准备过程
 	* @params mhdr 可以理解为一个镜像文件的imageLoadAddress(其实就是一个类的内存信息)
 	*/
void prepare_load_methods(const headerType *mhdr)
{
    size_t count, i;
    
    runtimeLock.assertWriting();
    
    classref_t *classlist =
    _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
    	//remapClass方法是获取类对应的指针
        schedule_class_load(remapClass(classlist[i]));
    }
    
    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        realizeClass(cls);
        assert(cls->ISA()->isRealized());
        add_category_to_loadable_list(cat);
    }
}
	```
    经过上述方法，会把需要调用+load方法的类添加到一个列表中，后续调用的时候回直接从该列表取出来。通过_getObjc2NonlazyClassList方法，获取到当前镜像中所有的类列表classlist。循环该数组，并通过remapClass方法获取到该类的对应的指针，然后通过调用schedule_class_load方法，递归地安排当前类和没有调用的+load父类进入列表。下面的categorylist原理一样，只是将类的category加入到列表中。
    schedule_class_load方法具体实现如下：
    
    ```
	static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());

    if (cls->data()->flags & RW_LOADED) return;

    schedule_class_load(cls->superclass);

    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED);
}
	```
    
    add_class_to_loadable_list将该类加入到一个全局的列表中，在此之前会将该类的父类调用add_class_to_loadable_list方法，确保父类在子类前面、子类在category的前面。
    注释：调用时，会优先调用当前继承线调用，

- +load方法的调用阶段
	+load方法准备完成之后，会调用call_load_methods方法，开始执行+load方法。call_load_methods方法具体实现如下：
    ```
    void call_load_methods(void)
{
    ...

    do {
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        more_categories = call_category_loads();

    } while (loadable_classes_used > 0  ||  more_categories);

    ...
}

	```

    类中的call_class_loads会从上述列表中找到对应的类，然后找到该类对应的+load方法的实现，然后调用。call_class_loads的具体实现如下：
    ```
    static void call_class_loads(void)
{
    int i;

    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;

    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue;
		//下面这行代码就会执行load方法
        (*load_method)(cls, SEL_load);
    }

    if (classes) free(classes);
}

	```
- +load方法加载的管理
	OC对于加载的管理主要使用了两个列表：loadable_classes和loadable_categories。方法的调用过程也分为两部分：准备laod和调用load；
    #####往列表中添加类(生产)：
    在调用：load_images -> load_images_nolock -> prepare_load_methods -> schedule_class_load -> add_class_to_loadable_list 的时候会将未加载的类添加到 loadable_classes 数组中：
    ```
	void add_class_to_loadable_list(Class cls)
{
    IMP method;

    loadMethodLock.assertLocked();
	//获取该类的+load方法
    method = cls->getLoadMethod();
    //判空
    if (!method) return;

	//判断loadable_classes这个数组列表是否完全被占用，如果完全被占用了，则增加长度。
    if (loadable_classes_used == loadable_classes_allocated) {
        loadable_classes_allocated = loadable_classes_allocated*2 + 16;
        loadable_classes = (struct loadable_class *)
            realloc(loadable_classes,
                              loadable_classes_allocated *
                              sizeof(struct loadable_class));
    }
	//将该类加入到列表
    loadable_classes[loadable_classes_used].cls = cls;
    //将该类的+load方法添加到列表
    loadable_classes[loadable_classes_used].method = method;
    //数量增加
    loadable_classes_used++;
}
	```
    分类的列表，代码如下，几乎和类的一样：
    ```
	void add_category_to_loadable_list(Category cat)
{
    IMP method;

    loadMethodLock.assertLocked();

    method = _category_getLoadMethod(cat);

    if (!method) return;

    if (loadable_categories_used == loadable_categories_allocated) {
        loadable_categories_allocated = loadable_categories_allocated*2 + 16;
        loadable_categories = (struct loadable_category *)
            realloc(loadable_categories,
                              loadable_categories_allocated *
                              sizeof(struct loadable_category));
    }

    loadable_categories[loadable_categories_used].cat = cat;
    loadable_categories[loadable_categories_used].method = method;
    loadable_categories_used++;
}
	```
	#####从列表中调用类(消费)：
    当类的call_class_loads调用 load 方法的过程就是“消费” loadable_classes 的过程，load_images -> call_load_methods -> call_class_loads 会从 loadable_classes 中取出对应类和方法，执行 load：

	```
	do {
    	while (loadable_classes_used > 0) {
        	call_class_loads();
    	}

    	more_categories = call_category_loads();

	} while (loadable_classes_used > 0  ||  more_categories);

	```
    调用顺序：
    1，不停调用类的 + load 方法，直到 loadable_classes 为空
	2，调用一次 call_category_loads 加载分类
	3，如果有 loadable_classes 或者更多的分类，继续调用 load 方法
    
    分类中load方法的调用：
    ```
    static bool call_category_loads(void)
{
    int i, shift;
    bool new_categories_added = NO;
    // 1. 获取当前可以加载的分类列表
    struct loadable_category *cats = loadable_categories;
    int used = loadable_categories_used;
    int allocated = loadable_categories_allocated;
    loadable_categories = nil;
    loadable_categories_allocated = 0;
    loadable_categories_used = 0;

    for (i = 0; i < used; i++) {
        Category cat = cats[i].cat;
        load_method_t load_method = (load_method_t)cats[i].method;
        Class cls;
        if (!cat) continue;

        cls = _category_getClass(cat);
        if (cls  &&  cls->isLoadable()) {
            // 2. 如果当前类是可加载的 `cls  &&  cls->isLoadable()` 就会调用分类的 load 方法
            (*load_method)(cls, SEL_load);
            cats[i].cat = nil;
        }
    }

    // 3. 将所有加载过的分类移除 `loadable_categories` 列表
    shift = 0;
    for (i = 0; i < used; i++) {
        if (cats[i].cat) {
            cats[i-shift] = cats[i];
        } else {
            shift++;
        }
    }
    used -= shift;

    // 4. 为 `loadable_categories` 重新分配内存，并重新设置它的值
    new_categories_added = (loadable_categories_used > 0);
    for (i = 0; i < loadable_categories_used; i++) {
        if (used == allocated) {
            allocated = allocated*2 + 16;
            cats = (struct loadable_category *)
                realloc(cats, allocated *
                                  sizeof(struct loadable_category));
        }
        cats[used++] = loadable_categories[i];
    }

    if (loadable_categories) free(loadable_categories);

    if (used) {
        loadable_categories = cats;
        loadable_categories_used = used;
        loadable_categories_allocated = allocated;
    } else {
        if (cats) free(cats);
        loadable_categories = nil;
        loadable_categories_used = 0;
        loadable_categories_allocated = 0;
    }

    return new_categories_added;
}
	```
    具体步骤：
    1，获取当前可以加载的分类列表
	2，如果当前类是可加载的 cls  &&  cls->isLoadable() 就会调用分类的 load 方法
	3，将所有加载过的分类移除 loadable_categories 列表
	4，为 loadable_categories 重新分配内存，并重新设置它的值
- 调用顺序规则：
	1，父类load方法先于子类调用
    2，类的load方法先于分类调用
    
    对于第一条规则，从源码中可以看到以下代码：
    ```
    static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());

    if (cls->data()->flags & RW_LOADED) return;

    schedule_class_load(cls->superclass);

    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED);
}
	```
    其中```schedule_class_load(cls->superclass)```方法，总能保证父类先于子类加入loadable_classes数组中，从而保证调用顺序的正确性。
    
    第二条规则，在源码中的处理主要在call_load_methods中实现：
    ```
    do {
    	while (loadable_classes_used > 0) {
        	call_class_loads();
    	}

    	more_categories = call_category_loads();

	} while (loadable_classes_used > 0  ||  more_categories);
	```
    上面的 do while语句能够在一定程度上确保类的laod方法能够先于分类的调用，但是不能完全保证正确性，如果分类的镜像在类的镜像之前加载到内存，上面的代码就不能保证类的load先于分类的调用了，所以在分类的load方法调用时，需要判断分类的类是否已经加载到内中去：
    ```
	if (cls  &&  cls->isLoadable()) {
    (*load_method)(cls, SEL_load);
    cats[i].cat = nil;
}
	```
    这里判断类是否存在，并且是否可以加载，如果都为真，则允许调用分类的+load方法。
    
    笔记来源：[原文链接](http://)
    
    #####完
    
