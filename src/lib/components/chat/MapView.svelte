<script>
    import { onMount } from 'svelte';
    import FloatingPanel from './FloatingPanel.svelte';
    
    // 可以添加任何需要的属性
    export let initialCoordinates = [39.9042, 116.4074]; // 默认坐标（北京）
    export let zoom = 13; // 默认缩放级别
    
    // 地图实例引用
    let map;
    let marker;
    let coordinates = "Loading...";
    let lastUpdated = new Date().toLocaleString();
    
    // 浮动窗口状态
    let isPanelVisible = false;
    
    // 引用父组件中的主内容区
    let mainContentElement = null;
    let mapContainer;
    
    // 位置信息内容 - 将传递给浮动面板
    $: locationInfo = {
        title: "位置信息",
        coordinates,
        lastUpdated
    };
    
    onMount(() => {
        // 尝试找到主内容区并保存引用
        setTimeout(() => {
            mainContentElement = document.querySelector('#chat-container > .flex-1') || 
                                document.querySelector('.flex-1.min-w-0.flex.flex-col') ||
                                document.querySelector('.flex-1');
            console.log("找到主内容区:", mainContentElement);
            
            // 初始化地图的函数（在找到主内容区后初始化地图）
            initMap();
        }, 100);
        
        // 清理函数 - 当组件卸载时执行
        return () => {
            if (map) {
                map.remove();
            }
        };
    });
    
    function initMap() {
        // 等待Leaflet脚本加载完成
        setTimeout(() => {
            if (typeof L !== 'undefined' && mapContainer) {
                // 初始化地图，设置视图中心和缩放级别
                map = L.map(mapContainer).setView([initialCoordinates[0], initialCoordinates[1]], zoom);
                
                // 添加OSM图层 - 这是一个免费且无需密钥的地图服务
                L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                    maxZoom: 19,
                    attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
                }).addTo(map);
                
                // 添加一个标记
                marker = L.marker([initialCoordinates[0], initialCoordinates[1]]).addTo(map);
                marker.bindPopup("<b>当前位置</b>").openPopup();
                
                // 显示坐标
                coordinates = `${initialCoordinates[0].toFixed(4)}, ${initialCoordinates[1].toFixed(4)}`;
                
                // 当用户点击地图时更新坐标
                map.on('click', function(e) {
                    const lat = e.latlng.lat.toFixed(4);
                    const lng = e.latlng.lng.toFixed(4);
                    coordinates = `${lat}, ${lng}`;
                    lastUpdated = new Date().toLocaleString();
                    
                    // 移动标记到新位置
                    marker.setLatLng(e.latlng);
                    marker.bindPopup(`<b>新位置</b><br>坐标: ${lat}, ${lng}`).openPopup();
                });
                
                // 解决地图可能无法正确渲染的问题
                setTimeout(() => {
                    map.invalidateSize();
                }, 100);
            }
        }, 500);
    }
    
    // 切换浮动面板的显示状态
    function togglePanel() {
        isPanelVisible = !isPanelVisible;
    }
</script>

<div class="map-component w-full h-full flex flex-col">
    <div class="flex justify-between items-center mb-4">
        <h2 class="text-xl font-bold">地图视图</h2>
        <button 
            class="bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded-md"
            on:click={togglePanel}
        >
            显示信息
        </button>
    </div>
    
    <!-- 地图容器 - 使用 flex-grow 占满剩余空间 -->
    <div bind:this={mapContainer} class="w-full flex-grow rounded-lg"></div>
    
    <!-- 引入浮动面板组件，传递主内容区引用 -->
    {#if isPanelVisible}
        <FloatingPanel 
            info={locationInfo} 
            on:close={() => isPanelVisible = false}
            fullscreen={true}
            mainContentRef={mainContentElement}
        />
    {/if}
</div>

<svelte:head>
    <!-- Leaflet CSS -->
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
        integrity="sha256-p4NxAoJBhIIN+hmNHrzRCf9tD/miZyoHS5obTRR9BMY="
        crossorigin=""/>
    
    <!-- Leaflet JavaScript -->
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
        integrity="sha256-20nQCchB9co0qIjJZRGuk2/Z9VM+kNiyxNV1lvTlZBo="
        crossorigin=""></script>
</svelte:head>

<style>
    /* 确保地图组件和容器占满全高 */
    .map-component {
        display: flex;
        flex-direction: column;
        height: 100%;
    }
    
    /* 确保地图正确显示 */
    :global(.leaflet-container) {
        z-index: 1;
        height: 100%;
        min-height: 200px; /* 设置最小高度，防止过小 */
    }
</style>