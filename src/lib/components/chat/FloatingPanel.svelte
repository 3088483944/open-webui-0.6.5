<script>
    import { createEventDispatcher, onMount } from 'svelte';
    
    // 创建事件分发器
    const dispatch = createEventDispatcher();
    
    // 组件属性
    export let info = {}; // 要显示的信息对象
    export let fullscreen = false; // 是否覆盖整个主内容区
    export let mainContentRef = null; // 允许从外部传入对主内容区的引用
    
    // 面板引用
    let panelElement;
    let isPositioned = false;
    let resizeObserver;
    let initialRender = true; // 添加一个标记来控制初始渲染
    
    onMount(() => {
        // 组件挂载后先隐藏面板，防止闪烁
        if (fullscreen && panelElement) {
            panelElement.style.visibility = 'hidden';
        }
        
        // 在组件挂载后延迟执行定位
        setTimeout(() => {
            positionPanel();
            // 定位完成后再显示面板
            if (fullscreen && panelElement) {
                panelElement.style.visibility = 'visible';
            }
            initialRender = false;
        }, 50);
        
        // 添加窗口大小变化的监听器
        window.addEventListener('resize', positionPanel);
        
        // 设置ResizeObserver来监听主内容区大小变化
        setupResizeObserver();
        
        // 清理函数
        return () => {
            window.removeEventListener('resize', positionPanel);
            if (resizeObserver) {
                resizeObserver.disconnect();
            }
        };
    });
    
    function setupResizeObserver() {
        if (typeof ResizeObserver !== 'undefined' && fullscreen) {
            resizeObserver = new ResizeObserver((entries) => {
                // 当观察到的元素大小变化时，重新定位面板
                positionPanel();
            });
            
            // 尝试观察主内容区
            setTimeout(() => {
                try {
                    const mainContentArea = findMainContentArea();
                    if (mainContentArea) {
                        resizeObserver.observe(mainContentArea);
                        console.log('ResizeObserver已设置');
                    }
                } catch (err) {
                    console.error('设置ResizeObserver时出错:', err);
                }
            }, 200);
        }
    }
    
    function findMainContentArea() {
        // 首先尝试使用传入的引用
        if (mainContentRef) return mainContentRef;
        
        // 更新选择器列表，优先使用更精确的选择器定位主内容区
        const selectors = [
            // 首先尝试找到最准确的主内容区选择器
            '#chat-container > .flex-1',
            '#chat-container > div.flex-1',
            '.flex-1.min-w-0.flex.flex-col',
            // 更通用的备选选择器
            '.flex-1',
            '#chat-container > div:first-child',
            '.min-w-0',
            '[role="main"]',
            'main',
            '.main-content'
        ];
        
        for (const selector of selectors) {
            const element = document.querySelector(selector);
            if (element && isValidElement(element)) {
                console.log(`找到主内容区: ${selector}`);
                return element;
            }
        }
        
        // 如果找不到，尝试使用更智能的查找方法
        const chatContainer = document.getElementById('chat-container');
        if (chatContainer) {
            // 查找chat-container中的第一个flex-1子元素
            const flexChildren = chatContainer.querySelectorAll('.flex-1');
            for (const child of flexChildren) {
                if (isValidElement(child)) {
                    console.log('通过chat-container找到flex-1子元素');
                    return child;
                }
            }
            
            // 如果依然找不到，尝试直接使用第一个子元素
            const firstChild = chatContainer.children[0];
            if (firstChild && isValidElement(firstChild)) {
                console.log('使用chat-container的第一个子元素');
                return firstChild;
            }
        }
        
        console.log('未找到主内容区');
        return null;
    }
    
    function isValidElement(element) {
        if (!element) return false;
        
        const rect = element.getBoundingClientRect();
        // 检查元素是否有合理的尺寸
        return rect.width > 100 && rect.height > 100;
    }
    
    function positionPanel() {
        if (!fullscreen || !panelElement) return;
        
        try {
            // 查找主内容区
            const mainContentArea = findMainContentArea();
            
            if (mainContentArea) {
                // 获取主内容区的位置和尺寸
                const rect = mainContentArea.getBoundingClientRect();
                
                // 确保位置和尺寸是合理的
                if (rect.width > 100 && rect.height > 100) {
                    // 应用样式到面板元素，使其完全覆盖主内容区
                    panelElement.style.position = 'fixed';
                    panelElement.style.top = `${rect.top}px`;
                    panelElement.style.left = `${rect.left}px`;
                    panelElement.style.width = `${rect.width}px`;
                    panelElement.style.height = `${rect.height}px`;
                    panelElement.style.zIndex = '50';
                    panelElement.style.overflow = 'auto';
                    
                    isPositioned = true;
                    console.log('面板已定位到主内容区:', rect);
                } else {
                    console.log('获取到的矩形尺寸过小:', rect);
                    fallbackPosition();
                }
            } else {
                console.log('未找到主内容区');
                fallbackPosition();
            }
        } catch (error) {
            console.error('定位面板时出错:', error);
            fallbackPosition();
        }
    }
    
    // 改进的备用定位方案
    function fallbackPosition() {
        if (panelElement && fullscreen) {
            // 查找Chat容器
            const chatContainer = document.getElementById('chat-container');
            if (chatContainer) {
                const containerRect = chatContainer.getBoundingClientRect();
                // 假设地图区占据1/3宽度，主内容区占据2/3宽度
                const mainContentWidth = containerRect.width * 2/3;
                
                // 应用样式覆盖主内容区
                panelElement.style.position = 'fixed';
                panelElement.style.top = `${containerRect.top}px`;
                panelElement.style.left = `${containerRect.left}px`;
                panelElement.style.width = `${mainContentWidth}px`;
                panelElement.style.height = `${containerRect.height}px`;
                panelElement.style.zIndex = '50';
                panelElement.style.overflow = 'auto';
                
                isPositioned = true;
                console.log('使用改进的备用定位:', { 
                    top: containerRect.top, 
                    left: containerRect.left, 
                    width: mainContentWidth, 
                    height: containerRect.height 
                });
                return;
            }
            
            // 如果还是找不到合适的定位，使用视口尺寸的估计值
            const viewportHeight = window.innerHeight;
            const viewportWidth = window.innerWidth;
            
            // 估计页面头部的高度
            const estimatedHeaderHeight = 60;
            
            // 估计内容区域的尺寸和位置
            let top = estimatedHeaderHeight;
            let left = 0;
            let width = viewportWidth * 0.7; // 约为视口宽度的70%
            let height = viewportHeight - estimatedHeaderHeight;
            
            // 应用样式
            panelElement.style.position = 'fixed';
            panelElement.style.top = `${top}px`;
            panelElement.style.left = `${left}px`;
            panelElement.style.width = `${width}px`;
            panelElement.style.height = `${height}px`;
            panelElement.style.zIndex = '50';
            panelElement.style.overflow = 'auto';
            
            isPositioned = true;
            console.log('使用视口大小的备用定位:', { top, left, width, height });
        }
    }
    
    // 关闭面板的方法
    function closePanel() {
        dispatch('close');
    }
    
    // 生成主题轨迹报告
    // 生成主题轨迹报告
function generateReport(institutionName, promptId) {
    // 获取当前信息
    const institution = institutionName || info.institutionName || "未指定机构";
    const id = promptId || info.promptId || "xxxx";
    const prompt = `请你生成${institution}的主题轨迹报告,id=${id}`;
    
    // 执行一系列自动化操作
    setTimeout(() => {
        // 1. 关闭当前浮动窗口
        closePanel();
        
        // 2. 选择"机构主题轨迹报告生成"模型
        setTimeout(() => {
            try {
                // 尝试点击模型选择器按钮（根据实际DOM结构调整选择器）
                const modelSelectorButton = document.querySelector('#model-selector-0-button') || 
                                          document.querySelector('[aria-label="选择一个模型"]') ||
                                          document.querySelector('[data-menu-trigger]') ||
                                          document.querySelector('[data-melt-dropdown-menu-trigger]');
                
                if (modelSelectorButton) {
                    modelSelectorButton.click();
                    console.log('点击了模型选择器按钮');
                    
                    // 等待模型选择菜单出现
                    setTimeout(() => {
                        // 查找"机构主题轨迹报告生成"选项
                        const reportModelOption = Array.from(document.querySelectorAll('button[data-value]')).find(el => {
                            return el.textContent.includes('机构主题轨迹报告生成') || el.getAttribute('data-value') === 'test';
                        });
                        
                        if (!reportModelOption) {
                            // 如果找不到，尝试更广泛的选择器
                            const allButtons = Array.from(document.querySelectorAll('button'));
                            const reportButton = allButtons.find(btn => 
                                btn.textContent.includes('机构主题轨迹报告生成') || 
                                btn.textContent.includes('主题轨迹') ||
                                btn.getAttribute('data-arrow-selected') === 'true'
                            );
                            
                            if (reportButton) {
                                reportButton.click();
                                console.log('已选择"机构主题轨迹报告生成"模型');
                            } else {
                                console.log('未找到"机构主题轨迹报告生成"模型选项');
                                // 记录所有按钮以便调试
                                console.log('可用按钮:', allButtons.map(b => b.textContent));
                            }
                        } else {
                            reportModelOption.click();
                            console.log('已选择"机构主题轨迹报告生成"模型');
                        }
                        
                        // 3. 输入prompt到输入框
                        setTimeout(() => {
                            const inputField = document.querySelector('textarea[placeholder]') || 
                                             document.querySelector('.textarea') ||
                                             document.querySelector('[contenteditable="true"]');
                            
                            if (inputField) {
                                // 设置输入框的值
                                if (inputField.tagName.toLowerCase() === 'textarea') {
                                    inputField.value = prompt;
                                } else {
                                    // 如果是contenteditable元素
                                    inputField.textContent = prompt;
                                }
                                
                                // 触发input事件，确保框架能检测到值的变化
                                const inputEvent = new Event('input', { bubbles: true });
                                inputField.dispatchEvent(inputEvent);
                                
                                // 也触发change事件
                                const changeEvent = new Event('change', { bubbles: true });
                                inputField.dispatchEvent(changeEvent);
                                
                                console.log('已在输入框中填入Prompt:', prompt);
                                
                                // 4. 自动点击发送按钮
                                setTimeout(() => {
                                    // 查找发送按钮
                                    const sendButton = document.querySelector('button[aria-label="Send message"]') ||
                                                     document.querySelector('button[type="submit"]') ||
                                                     Array.from(document.querySelectorAll('button')).find(btn => {
                                                         return btn.textContent.includes('发送') || 
                                                                btn.textContent.includes('Send') ||
                                                                btn.querySelector('svg[data-icon="paper-plane"]') ||
                                                                btn.classList.contains('send-button');
                                                     });
                                    
                                    if (sendButton && !sendButton.disabled) {
                                        sendButton.click();
                                        console.log('已点击发送按钮');
                                    } else {
                                        console.log('未找到发送按钮或按钮被禁用');
                                        // 记录所有可能的按钮
                                        console.log('可用按钮:', Array.from(document.querySelectorAll('button')).map(b => ({
                                            text: b.textContent,
                                            classes: b.className,
                                            disabled: b.disabled
                                        })));
                                    }
                                }, 800);
                            } else {
                                console.log('未找到输入框');
                                // 尝试查找并记录所有可能的输入元素
                                const allInputs = [
                                    ...Array.from(document.querySelectorAll('textarea')),
                                    ...Array.from(document.querySelectorAll('[contenteditable="true"]')),
                                    ...Array.from(document.querySelectorAll('input[type="text"]'))
                                ];
                                console.log('可能的输入元素:', allInputs);
                            }
                        }, 800);
                    }, 800);
                } else {
                    console.log('未找到模型选择器按钮');
                    // 记录页面上的按钮以便调试
                    const allButtons = Array.from(document.querySelectorAll('button'));
                    console.log('页面上的按钮:', allButtons.map(b => ({
                        id: b.id,
                        text: b.textContent,
                        ariaLabel: b.getAttribute('aria-label'),
                        classes: b.className
                    })));
                }
            } catch (error) {
                console.error('自动化流程执行出错:', error);
            }
        }, 500);
    }, 100);
}
</script>

{#if fullscreen}
    <!-- 覆盖主内容区的面板 -->
    <div 
        bind:this={panelElement}
        class="bg-white dark:bg-gray-800 rounded overflow-auto shadow-lg"
        style="visibility: {initialRender ? 'hidden' : 'visible'};"
    >
        <!-- 内容容器 -->
        <div class="p-6 h-full flex flex-col">
            <!-- 顶部标题栏 -->
            <div class="flex justify-between items-center mb-6">
                <h3 class="text-xl font-bold text-gray-800 dark:text-gray-200">
                    {info.title || '位置信息'}
                </h3>
                
                <!-- 关闭按钮 -->
                <button 
                    class="text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200"
                    on:click={closePanel}
                >
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                    </svg>
                </button>
            </div>
            
            <!-- 内容区域 - 可扩展为多个插槽 -->
            <div class="flex-grow text-gray-600 dark:text-gray-300 overflow-y-auto">
                <!-- 位置信息显示 -->
                {#if info.coordinates}
                    <div class="mb-4">
                        <p class="text-sm mb-1">机构名: Wuhan University</p>
                        <p class="text-base font-medium">坐标: <span>{info.coordinates}</span></p>
                        
                        <!-- 生成报告按钮 -->
                        <button 
                            class="mt-4 bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded-md"
                            on:click={() => generateReport("武汉大学")}
                        >
                            生成主题轨迹报告
                        </button>
                    </div>
                {/if}
                
                <!-- 时间戳信息 -->
                {#if info.lastUpdated}
                    <p class="text-xs text-gray-500 dark:text-gray-400 mt-4">
                        最后更新: {info.lastUpdated}
                    </p>
                {/if}
                
                <!-- 自定义内容插槽 -->
                <slot></slot>
            </div>
            
            <!-- 底部按钮区域 -->
            <div class="mt-6 flex justify-end">
                <slot name="actions">
                    <button 
                        class="bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded-md"
                        on:click={closePanel}
                    >
                        关闭
                    </button>
                </slot>
            </div>
        </div>
    </div>
{:else}
    <!-- 常规居中浮动面板 -->
    <div 
        class="fixed inset-0 z-50 flex items-center justify-center"
        style="background-color: rgba(0, 0, 0, 0.5);"
    >
        <div class="bg-white dark:bg-gray-800 rounded-lg p-6 relative overflow-auto w-80">
            <!-- 关闭按钮 -->
            <button 
                class="absolute top-4 right-4 text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200"
                on:click={closePanel}
            >
                <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                </svg>
            </button>
            
            <!-- 标题区域 -->
            <h3 class="text-xl font-bold mb-6 text-gray-800 dark:text-gray-200">
                {info.title || '信息面板'}
            </h3>
            
            <!-- 内容区域 -->
            <div class="text-gray-600 dark:text-gray-300">
                <!-- 位置信息显示 -->
                {#if info.coordinates}
                    <div class="mb-4">
                        <p class="text-sm mb-1">当前显示位置数据</p>
                        <p class="text-base font-medium">坐标: <span>{info.coordinates}</span></p>
                    </div>
                {/if}
                
                <!-- 时间戳信息 -->
                {#if info.lastUpdated}
                    <p class="text-xs text-gray-500 dark:text-gray-400 mt-4">
                        最后更新: {info.lastUpdated}
                    </p>
                {/if}
                
                <!-- 自定义内容插槽 -->
                <slot></slot>
            </div>
            
            <!-- 底部按钮区域 -->
            <div class="mt-6 flex justify-end">
                <slot name="actions">
                    <button 
                        class="bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded-md"
                        on:click={closePanel}
                    >
                        关闭
                    </button>
                </slot>
            </div>
        </div>
    </div>
{/if}