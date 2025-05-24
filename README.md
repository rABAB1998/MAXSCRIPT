# MAXSCRIPT
3DMAXAPI

--// 3ds Max API函数集

-- 创建Bone骨骼对象
-- @param boneName: 骨骼名称  
-- @param startPos: 起始位置坐标 [x,y,z]
-- @param endPos: 结束位置坐标 [x,y,z]  
-- @param zAxis: (内部参数)骨骼轴向，默认Z轴向上 [0,0,1]
-- @return: 返回新建的骨骼对象
-- 注：创建的骨骼默认启用鳍状显示(fins)
fn createBone boneName startPos endPos =
(
    local zAxis = [0, 0, 1]  -- 骨骼局部坐标系Z轴方向
    local newBone = boneSys.createBone startPos endPos zAxis
    newBone.name = boneName
    return newBone
)

-- 创建点辅助对象(Point Helper)
-- @param pointName: 名称(需唯一)  
-- @param position: 世界坐标位置 [x,y,z]
-- @param shape: 显示形状枚举:
--   #cross: 十字形 (默认)
--   #box: 方框
--   #axis: 三轴指示器  
--   #center: 中心标记
-- @param size: 显示尺寸(单位: 3ds Max系统单位)
-- @note: 会先删除同名的现有对象
fn createPoint pointName position shape:#cross size:20.0 =
(
    local newPoint = point pos:position
    newPoint.name = pointName
    
    -- 重置所有显示属性确保互斥
    newPoint.cross = off
    newPoint.Box = off
    newPoint.axistripod = off
    newPoint.centermarker = off
    
    -- 根据shape参数激活对应显示样式
    case shape of
    (
        #cross: newPoint.cross = on
        #box: newPoint.Box = on
        #axis: newPoint.axistripod = on
        #center: newPoint.centermarker = on
        default: 
        (
            format "警告：未知形状类型%，默认使用十字形\n" shape
            newPoint.cross = on
        )
    )
    
    -- 设置显示尺寸
    newPoint.size = size
    
    return newPoint
)


-- 创建圆形控制器
-- @param circleName: 圆形名称
-- @param centerPos: 圆心位置
-- @param color: 线框颜色
-- @return: 新建的圆形对象
fn createCircle circleName centerPos color =
(
    local circleObj = circle radius:10 pos:centerPos
    circleObj.wireColor = color
    circleObj.name = circleName
    return circleObj
)


-- 创建自定义样条线(Spline)
-- @param lineName: 样条线名称  
-- @param posList: 控制点坐标数组(至少需要2个点)
-- @param lineColor: 线框颜色，默认淡紫色
-- @param knotType: 节点类型:
--   #smooth: 平滑节点 (默认) 
--   #corner: 直角节点
-- @param segmentType: 线段类型:
--   #curve: 曲线段 (默认)
--   #line: 直线段
-- @return: 返回可编辑样条对象
-- @warning: 会自动转换为可编辑样条(Editable Spline)
fn createLineShape lineName posList lineColor:(color 229 154 215) knotType:#smooth segmentType:#curve = 
(
    -- 1. 创建基础样条线对象
    local splineObj = line()
    
    -- 2. 添加新样条
    local splineIndex = addNewSpline splineObj
    
    -- 3. 添加所有控制点
    for pos in posList do
    (
        addKnot splineObj splineIndex knotType segmentType pos
    )
    
    -- 4. 更新样条显示
    updateShape splineObj
    
    -- 5. 转换为可编辑样条
    convertToSplineShape splineObj
    
    -- 6. 设置属性
    splineObj.wireColor = lineColor
    splineObj.name = lineName
    
    -- 7. 返回创建的对象
    return splineObj
)


-- 父子链接
-- @param parentObj: 父对象
-- @param childObj: 子对象
fn mySetParent  parentObj childObj =
(
    childObj.parent = parentObj
)

-- 链接约束
-- @param sourceObj: 源对象（父对象）
-- @param targetObj: 目标对象（子对象）
fn addLinkConstraint sourceObj targetObj =
(
    targetObj.Transform.controller = Link_Constraint()
    targetObj.transform.controller.AddTarget sourceObj 0
)

-- 注视约束
-- @param controlledObj: 需要旋转的对象(如相机)
-- @param targetObj: 注视目标
-- @param pickUpNode: 上方向控制对象(可选)
-- @配置细节:
--   - 强制使用rotation_list保证可扩展性
--   - 默认世界坐标系上方向
--   - 提供pickUpNode时自动切换为节点控制
--   - 锁定视线长度为0(不显示注视线)
fn addLookAtConstraint controlledObj targetObj pickUpNode:unsupplied =
(
    -- 创建旋转列表控制器
    controlledObj.rotation.controller = rotation_list()
    
    -- 添加LookAt约束
    local lookAtConstraint = LookAt_Constraint()
    controlledObj.rotation.controller[1].controller = lookAtConstraint
    
    -- 设置注视目标
    lookAtConstraint.appendTarget targetObj 100  -- 100%权重
    lookAtConstraint.lookat_vector_length = 0   -- 注视视线长度
    
    -- 智能处理上方向设置
    if pickUpNode == unsupplied then
    (
        -- 使用世界坐标系上方向
        lookAtConstraint.upnode_world = on
    )
    else
    (
        -- 使用指定对象控制上方向
        lookAtConstraint.upnode_world = off
        lookAtConstraint.pickUpNode = pickUpNode
    )
    
    lookAtConstraint.upnode_ctrl = 0 -- 上方向方式：注视or轴对齐
    lookAtConstraint.relative = off --保持偏移 

)



-- 旋转约束
-- @param targetObj: 要添加约束的对象
-- @param controlObjs: 控制对象数组，可以是单个对象或多个对象的数组
-- @param isRelative: 是否保持偏移 [默认:false]
-- @param weights: (可选)自定义权重数组
fn addRotationConstraint targetObj controlObjs isRelative:off weights:unsupplied =
(
    -- 确保controlObjs是数组形式
    local targets = if classOf controlObjs == Array then controlObjs else #(controlObjs)
    local targetCount = targets.count
    
    -- 准备权重数组
    local weightValues = if weights != unsupplied then weights else
    (
        local defaultWeight = 100.0 / targetCount
        for i=1 to targetCount collect defaultWeight
    )
    
    -- 设置控制器
    targetObj.rotation.controller = rotation_list()
    local rotConstraint = Orientation_Constraint()    
    targetObj.rotation.controller.Available.controller = rotConstraint
    
    -- 添加所有目标
    for i=1 to targetCount do
    (
        rotConstraint.appendTarget targets[i] weightValues[i]
    )
    
    rotConstraint.relative = isRelative
    
    -- 输出调试信息
    format "已为%添加%d个旋转约束目标\n" targetObj.name targetCount
)


-- 位置约束
-- @param targetObj: 要添加约束的对象
-- @param controlObjs: 控制对象数组，可以是单个对象或多个对象的数组
-- @param isRelative: 是否保持偏移 [默认:false]
-- @param weights: (可选)自定义权重数组
--
-- 使用示例：
-- addPositionConstraint point_01 point_02 isRelative:on -- 控制point_02跟随point_01(100%权重)
-- addPositionConstraint point_03 #(point_01, point_02) isRelative:on -- 控制point_03跟随point_01和point_02(各50%权重)
-- addPositionConstraint point_04 #(point_01, point_02, point_03) isRelative:on  -- 三目标均分权重+相对偏移

fn addPositionConstraint targetObj controlObjs isRelative:off weights:unsupplied =
(
    -- 确保controlObjs是数组形式
    local targets = if classOf controlObjs == Array then controlObjs else #(controlObjs)
    local targetCount = targets.count
    
    -- 准备权重数组
    local weightValues = if weights != unsupplied then weights else
    (
        local defaultWeight = 100.0 / targetCount
        for i=1 to targetCount collect defaultWeight
    )
    
    -- 设置控制器
    targetObj.pos.controller = position_list()
    local posConstraint = Position_Constraint()    
    targetObj.pos.controller.Available.controller = posConstraint
    
    -- 添加所有目标
    for i=1 to targetCount do
    (
        posConstraint.appendTarget targets[i] weightValues[i]
    )
    
    posConstraint.relative = isRelative
    
    -- 输出调试信息
    format "已为%添加%d个位置约束目标\n" targetObj.name targetCount
)


-- 路径约束
-- @param targetObjs: 对象数组(按数组顺序分布)
-- @param pathObj: 路径曲线对象  
-- @param isFollow: 是否跟随路径切线方向
-- @param isLoop: 是否循环路径
-- @param axis: 对齐轴向:
--   0: X轴对齐路径
--   1: Y轴对齐路径  
--   2: Z轴对齐路径(默认)
-- @note: 自动计算百分比使对象均匀分布
-- 使用示例:
--   -- 将4个点均匀约束到曲线上
--   addUniformPathConstraint #($Point1,$Point2,$Point3,$Point4) $MyPath
fn addUniformPathConstraint targetObjs pathObj isFollow:on isLoop:on axis:2 =
(
    -- 确保targetObjs是数组形式
    local targets = if classOf targetObjs == Array then targetObjs else #(targetObjs)
    local targetCount = targets.count
    
    -- 计算每个对象的路径百分比
    local percentStep = 100.0 / (targetCount - 1)
    
    for i = 1 to targetCount do
    (
        local targetObj = targets[i]
        local percent = if i == 1 then 0 else (percentStep * (i-1))
        
        -- 1. 创建位置列表控制器
        targetObj.pos.controller = position_list()
        
        -- 2. 添加路径约束到列表控制器
        local pathConstraint = Path_Constraint()
        targetObj.pos.controller.Available.controller = pathConstraint
        
        -- 3. 设置路径和基本参数
        pathConstraint.path = pathObj
        pathConstraint.follow = isFollow
        pathConstraint.loop = isLoop
        pathConstraint.axis = axis
        
        -- 4. 设置路径百分比
        pathConstraint.percent = percent
        
        -- 5. 输出调试信息
        format "已为%添加路径约束(位置%%)，路径对象为%\n" targetObj.name percent "%" pathObj.name
    )
    
    return true
)

-- =============================================
-- 工具集设计规范:
-- 1. 创建函数:
--   - 始终返回创建的对象
--   - 自动处理命名冲突(删除同名对象)
--   - 提供合理的默认参数
--
-- 2. 约束函数:
--   - 参数顺序: 被控制对象在前，目标对象在后
--   - 支持多目标输入(数组形式)
--   - 布尔参数使用is前缀(如isRelative)
--
-- 3. 错误处理:
--   - 关键操作添加format输出日志
--   - 可选参数使用unsupplied作为默认值
-- =============================================

-- 测试环境初始化函数
fn initTestScene = 
(
    -- 1. 清空场景
    resetMaxFile #noPrompt  -- 无提示重置
    
    -- 2. 创建基础测试对象
    global point_01 = createPoint "point_01" [0,0,0] shape:#box
    global point_02 = createPoint "point_02" [0,0,20] shape:#box
    global point_03 = createPoint "point_03" [0,10,0]
    global point_04 = createPoint "point_04" [0,20,0] shape:#box
    global line_01 = createLineShape "line_01" #( [0,0,0], [0,0,20], [0,10,0], [0,20,0] ) lineColor:(color 229 154 215) knotType:#smooth segmentType:#curve

)

-- 示例测试流程
fn testLookAtConstraint = 
(
    -- 初始化场景
    initTestScene()
    addUniformPathConstraint #(point_01, point_02, point_03, point_04) line_01
    -- addPathConstraint point_01 line_01 isFollow:on isLoop:on  -- 添加路径约束

    -- addPositionConstraint point_01 point_02 isRelative:on -- 控制point_02跟随point_01(100%权重)
    -- addPositionConstraint point_03 #(point_01, point_02) isRelative:on  -- 控制point_03跟随point_01和point_02(各50%权重)
    -- addPositionConstraint point_04 #(point_01, point_02, point_03) isRelative:on  -- 三目标均分权重+相对偏移

)

-- 运行测试
testLookAtConstraint()
