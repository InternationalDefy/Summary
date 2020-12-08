## VulkanHpp
- Namespace 
    - vkCreateInstance can be accessed as vk::createInstance
    - VkImageTiling can be accessed as vk::ImageTiling
    - VkImageCreateInfo can be accessed as vk::ImageCreateInfo
- Scoped Enums
    - VK_IMAGETYPE_2D is now vk::ImageType::e2D.
    - VK_COLOR_SPACE_SRGB_NONLINEAR_KHR is now vk::ColorSpaceKHR::eSrgbNonlinear.
    - VK_STRUCTURE_TYPE_PRESENT_INFO_KHR is now vk::StructureType::ePresentInfoKHR.
    - 移除_BIT后置
- Handles
    - vkBindBufferMemory(device, ...) -> device.bindBufferMemory(...) 
    - Handle拥有类型安全性
    - Handle的隐式类型转换支持 #define VULKAN_HPP_TYPESAFE_CONVERSION 0(32bit)/1(64bit) 
- Flags bitmask
    - 强类型Flag不再支持位运算, 但可以显式地使用EFlag类
    ```cpp
    vk::ImageUsageFlags iu1; // initialize a bitmask with no bit set
    vk::ImageUsageFlags iu2 = {}; // initialize a bitmask with no bit set
    vk::ImageUsageFlags iu3 = vk::ImageUsageFlagBits::eColorAttachment; // initialize with a single value
    vk::ImageUsageFlags iu4 = vk::ImageUsageFlagBits::eColorAttachment | vk::ImageUsageFlagBits::eStorage; // or two bits to get a bitmask
    PipelineShaderStageCreateInfo ci( {} /* pass a flag without any bits set */, ...);
    ```
- CreateInfo(builder) structs vs. Designate Initializers
    - C风格Vulkan
    ```c
    VkImageCreateInfo ci;
    ci.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
    ci.pNext = nullptr;
    ci.flags = ...some flags...;
    ci.imageType = VK_IMAGE_TYPE_2D;
    ci.format = VK_FORMAT_R8G8B8A8_UNORM;
    ci.extent = VkExtent3D { width, height, 1 };
    ci.mipLevels = 1;
    ci.arrayLayers = 1;
    ci.samples = VK_SAMPLE_COUNT_1_BIT;
    ci.tiling = VK_IMAGE_TILING_OPTIMAL;
    ci.usage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT;
    ci.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
    ci.queueFamilyIndexCount = 0;
    ci.pQueueFamilyIndices = 0;
    ci.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    vkCreateImage(device, &ci, allocator, &image));
    ```
    - VulkanHpp
    ```cpp
    vk::ImageCreateInfo ci({}, vk::ImageType::e2D, vk::Format::eR8G8B8A8Unorm,
                       { width, height, 1 },
                       1, 1, vk::SampleCountFlagBits::e1,
                       vk::ImageTiling::eOptimal, vk::ImageUsageFlagBits::eColorAttachment,
                       vk::SharingMode::eExclusive, 0, nullptr, vk::ImageLayout::eUndefined);
    vk::Image image = device.createImage(ci);
    ```
    - VulkanHpp(Cpp20 Designate Initilizers)
    ```cpp
    // initialize the vk::ApplicationInfo structure
    vk::ApplicationInfo applicationInfo{ .pApplicationName   = AppName,
                                        .applicationVersion = 1,
                                        .pEngineName        = EngineName,
                                        .engineVersion      = 1,
                                        .apiVersion         = VK_API_VERSION_1_1 };
            
    // initialize the vk::InstanceCreateInfo
    vk::InstanceCreateInfo instanceCreateInfo{ .pApplicationInfo = & applicationInfo };
    ```
- Array Proxy
    - 模板函数可以接受 临时变量数组, 数组的排列组合, std::array, std::vector.
    - 以下的例子可能有问题, 因为Scissor不仅牵扯到offset和size, 还有viewport
    - C-Vulkan
    ```C
    VkRect2D scissor{};
    scissor.offset = { 0, 0 };
    VkPipelineViewportStateCreateInfo viewportState{};		
    viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
    viewportState.viewportCount = 1;
    viewportState.pViewports = &viewport;
    viewportState.scissorCount = 1;
    viewportState.pScissors = &scissor;
    ```
    - Vulkan-Hpp
    ```cpp
    vk::CommandBuffer buffer;
	buffer.setScissor(0, { {1280, 720 } });
    ```
- Parsing Structs to Funciton
    - Vulkan-Hpp 可以将struct参数列表接受至Function中, 使用std::option<T> 填补空余的部分.
    - C-Vulkan
    ```C
    VkImageSubresource subResource;
    subResource.aspectMask = 0;
    subResource.mipLevel = 0;
    subResource.arrayLayer = 0;
    VkSubresourceLayout layout;
    vkGetImageSubresourceLayout(device, image, &subresource, &layout);
    ```
    - Vulkan-Hpp
    ```C++
    // flags here is a std::option<Flag>
    auto layout = device.getImageSubresourceLayout(image, { {} /* flags*/, 0 /* miplevel */, 0 /* arrayLayer */});
    ```
- Structure Pointer Chains
    - 链式数据的解决方案
    - 链上移除节点 vk::StructureChain::unlink<ClassType>(). -> remove ->  vk::StructureChain::relink<ClassType>().
    - 链是pNext的类型安全模板类实现
    ```C++
    // 模板展开
    vk::StructureChain<vk::MemoryAllocateInfo, vk::MemoryDedicatedAllocateInfo> c = {
        vk::MemoryAllocateInfo(size, type),
        vk::MemoryDedicatedAllocateInfo(image)
    };
    // 带类型检查的Get函数
    vk::StructureChain<vk::MemoryAllocateInfo, vk::ImportMemoryFdInfoKHR> c;
    vk::MemoryAllocateInfo &allocInfo = c.get<vk::MemoryAllocateInfo>();
    vk::ImportMemoryFdInfoKHR &fdInfo = c.get<vk::ImportMemoryFdInfoKHR>();
    // 查询
    auto result = device.getBufferMemoryRequirements2KHR<vk::MemoryRequirements2KHR, vk::MemoryDedicatedRequirementsKHR>({});
    vk::MemoryRequirements2KHR &memReqs = result.get<vk::MemoryRequirements2KHR>();
    vk::MemoryDedicatedRequirementsKHR &dedMemReqs = result.get<vk::MemoryDedicatedRequirementsKHR>();
    ```

- Return Values, Error Codes & Exceptions
    - C++ Features 在Vulkanhpp中的使用
    - 返回值可以是 vk::ResultValue<Value> 枚举类, 包含函数返回信息和返回值 类似于Rust Result<E,V>的处理
    - 在更新中, 函数会越来越倾向于使用 ResultValue<>, 在Cpp Api中可以查询具体的函数返回值
    - 以下几个示例
    - 使用C风格API
    ```C
    ShaderModuleCreateInfo createInfo(...);
    ShaderModule shader;
    Result result = device.createShaderModule(&createInfo, allocator, &shader);
    if (result.result != VK_SUCCESS)
    {
        handle error code;
    }
    ```
    - 使用std::tie(C++14)
    ```C++
    vk::Result result;
    vk::ShaderModule shaderModule;
    std::tie(result, shaderModule2)  = device.createShaderModule({...} /* createInfo temporary */);
    if (result != VK_SUCCESS)
    {
        handle error code;
    }
    ```
    - 使用structured binding(C++17)
    ```C++
    auto [result, shaderModule2] = device.createShaderModule({...} /* createInfo temporary */);
    if (result != VK_SUCCESS)
    {
        handle error code;
    }
    ```
    
- Enumeration
    - 在VulkanHpp中, C风格API对数组的循环遍历以enumerate函数取代, 返回值为std::vector
    ```C++  
    std::vector<LayerProperties> properties = physicalDevice.enumerateDeviceLayerProperties();
    ```

- UniqueHandle for automatic resource management
    - vk::UniqueHandle<Type, Deleter> 
    - 例如`vk::UniqueBuffer//vk::Buffer`,创建时使用`vk::Device::CreateBufferUnique//vk::Device::CreateBuffer`
    - 可以理解为UniquePtr

- Extensions / Per Device Function Pointers
    - 这部分说实话没看懂, 应该是跨平台渲染引擎使用的
    - Vulkan-Hpp只暴露了核心Features和一部分Extensions, 在需要对Extension做修改的情况下, 有需求暴露更多的Functions
    - DispatchLoaderDynamic 可以fetch所有的函数指针 
    ```C++
    // fetch all funciton pointers to scope
    vk::DispatchLoaderDynamic dldi(instance) 
    // Pass dispatch class to function call as last parameter
    device.getQueue(graphics_queue_family_index, 0, &graphics_queue, dldid);
    ```
- Type Traits
    - `template <typename EnumType, EnumType value> struct CppType` 
        - Maps `IndexType(trait)` values `(e.g. IndexType::eUint16, IndexType::eUint32, ...)` to the corresponding type `(e.g. uint16_t, uint32_t, ...)` by the member type `Type`; 
        - Maps `ObjectType(trait)` values `(e.g. ObjectType::eInstance, ObjectType::eDevice, ...)` to the corresponding type `(vk::Instance, vk::Device, ...)` by the member type `Type`; 
        - Maps `DebugReportObjectType(trait)` values `(e.g.DebugReportObjectTypeEXT::eInstance, DebuReportObjectTypeEXT::eDevice, ...)` to the corresponding type `(vk::Instance, vk::Device, ...)` by the member type `Type`;
    - `template <typename T> struct IndexTypeValue(trait)` Maps scalar types `(e.g. uint16_t, uint32_t, ...)` to the corresponding `IndexType` value `(IndexType::eUint16, IndexType::eUint32, ...)`.
    - `template <typename T> struct isVulkanHandleType` Maps a type to `true` **if and only if** it's a handle class `(vk::Instance, vk::Device, ...)` by the *static member* `value`.
    - `HandleClass::CType` Maps a handle class `(e.g. vk::Instance, vk::Device, ...)` to the corresponding C-type `(VkInstance, VkDevice, ...)` by the member type `CType`.
    - `HandleClass::objectType` Maps a handle class `(e.g. vk::Instance, vk::Device, ...)` to the corresponding `ObjectType` value `(ObjectType::eInstance, ObjectType::eDevice, ...)` by the static member `objectType`.
    - `HandleClass::debugReportObjectType` Maps a handle class `(e.g. vk::Instance, vk::Device, ...)` to the corresponding `DebugReportObjectTypeEXT` value `(DebugReportObjectTypeEXT::eInstance, DebugReportObjectTypeEXT::eDevice, ...)` by the static member `debugReportObjectType`.