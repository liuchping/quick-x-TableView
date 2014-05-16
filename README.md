quick-x-TableView
=================

实现拖动加载cell，移除单个cell


local kSelectSpriteTag = 2000

local LTableView = class("LTableView", function(rect)
    if not rect then rect = CCRect(0, 0, 0, 0) end
    local node = display.newClippingRegionNode(rect)
    node:setTouchEnabled(true)
    node:setCascadeBoundingBox(rect)
    require("framework.api.EventProtocol").extend(node)
    return node
end)

function LTableView:ctor( rect )
    self.touchRect = rect
    self.showNums = 0
    self.addNums = 0
    self.totalHeight = 0
    self.dragThreshold = 30
    self.cellSpacing = 0 --cell间距
    self.cells = {}

    self:registerScriptHandler(function(event)
        if event == "enter" then
            self:onEnter()
        elseif event == "exit" then
            self:onExit()
        end
    end)

    self.view = display.newNode()
    self:addChild(self.view)
    
    self.view:setTouchEnabled(true)
    self.view:addTouchEventListener(function(event, x, y)
        return self:onTouch(event, x, y)
    end)
end

function LTableView:onTouch( event, x, y )
    if event == "began" then
        if not self.touchRect:containsPoint(ccp(x, y)) then return false end
        return self:onTouchBegan(x, y)
    elseif event == "moved" then
        self:onTouchMoved(x, y)
    elseif event == "ended" then
        self:onTouchEnded(x, y)
    else -- cancelled
        self:onTouchCancelled(x, y)
    end
end

function LTableView:onTouchBegan( x, y )
    self.drag = {
        startX = x,
        startY = y,
        isTap = true,
        direction = false--false为向下拖拉，true为向上拖拉
    }
    return true
end

function LTableView:onTouchMoved( x, y )  
    if self.drag.isTap and math.abs(y - self.drag.startY) >= self.dragThreshold then
        if y - self.drag.startY > 0 then self.drag.direction = true end
        if self.getLabel and self.drag.direction then
            self.getLabel:setString("松开获取更多")
        end
        self.drag.isTap = false
    end
    if not self.drag.isTap then
        local offset = {
            x = 0,
            y = y - self.drag.startY
        }
        self.drag.startY = y
        self:setCellPos(offset)
    end
end

function LTableView:onTouchEnded( x, y )
    if self.drag.isTap then
        self:onTouchEndedWithTap(x, y)
    else
        self:onTouchEndedWithoutTap(x, y)
    end
    self.drag = nil
end
--判断cell是否被点击
function LTableView:onTouchEndedWithTap( x, y )
    local offx, offy = self.view:getPosition()
    local px, py = x - offx, y - offy

    for i = 1, self.showNums do
        local cell = self.cells[i]
        local xx, yy = cell:getPosition()
        local size = cell:getContentSize()
        local w = size.width
        local h = size.height
        local rect = CCRect(xx, yy, w, h)
        if rect:containsPoint(ccp(px, py)) then 
            if cell.selected then
                cell:selected()
            end
            if cell.onTap then
                cell:onTap()
            end
            self.currIndex = i
        else
            if cell.unselected then
                cell:unselected()
            end
        end
    end
end

function LTableView:onTouchEndedWithoutTap( x, y )
    local offset = {
        x = 0,
        y = y - self.drag.startY
    }
    self:setCellPos(offset)

    local _x_last, _y_last = self.cells[self.showNums]:getPosition()
    local _size_last = self.cells[self.showNums]:getContentSize()
    if self.drag.direction and self.showNums < self.totalCellNums and _y_last >= -(self.touchRect.size.height - _size_last.height + 20) then
        self:showCell()
        self.drag.direction = false
    end
    
    self:onTouchEndedOffsetManage(x, y)
end
--触摸结束后位置偏移处理
function LTableView:onTouchEndedOffsetManage( x, y )
    local _x_first, _y_first = self.cells[1]:getPosition()
    local _size_first = self.cells[1]:getContentSize()

    local offsetY_ani = 0
    if _y_first < -(_size_first.height + self.cellSpacing) then
        offsetY_ani = _y_first + _size_first.height + self.cellSpacing
    elseif _y_first > (_size_first.height + self.cellSpacing) * (self.totalCellNums - 2) then
        offsetY_ani = _y_first - (_size_first.height + self.cellSpacing) * (self.totalCellNums - 2)
    else 
        local m1, m2 = math.modf(math.abs(_y_first)/math.abs(_size_first.height + self.cellSpacing))
        if m2 > 0.5 then
            offsetY_ani = -(_size_first.height + self.cellSpacing) * (1 - m2)
        else
            offsetY_ani = (_size_first.height + self.cellSpacing) * m2         
        end
        if _y_first < 0 then
            if m2 > 0.5 then
                offsetY_ani = (_size_first.height + self.cellSpacing) * (1 - m2)
            else
                offsetY_ani = -(_size_first.height + self.cellSpacing) * m2       
            end
        end
    end
    self:cellMoveAni(offsetY_ani)
end
--添加cell
function LTableView:addCell( cell )
    self.view:addChild(cell, 1)
    self.cells[#self.cells + 1] = cell
end
--设置总的cell数量
function LTableView:setTotalCellNums( nums )
    self.view:setPosition(self.touchRect.origin.x , self.touchRect.origin.y + self.touchRect.size.height)
    dump(nums)
    self.totalCellNums = nums
end
--设置创建cell的回调
function LTableView:setCellCallBack( cell_cb )
    self.cell_cb = nil
    self.cell_cb = cell_cb
end
--按需求展示cell
function LTableView:showCell( )
    if self.showNums > 0 then
        local cell = self.cells[self.showNums]
        local cx, cy = cell:getPosition()
        self.totalHeight = -cy
    end
    local addNums = self:getAddNums()
    for i = 1, addNums do
        dump(self.showNums + i)
        self.cell_cb(self.showNums + i)
        local cell = self.cells[self.showNums + i]
        cell:setTag(self.showNums + i)

        local _size = cell:getContentSize()
        self.totalHeight = self.totalHeight + _size.height + self.cellSpacing
        --竖向列表
        local _x = 0
        local _y = - self.totalHeight
        cell:setAnchorPoint(ccp(0,0))
        cell:setPosition(_x, _y)
    end
    self.showNums = self.showNums + addNums
    --是否显示拖动获取
    self:showGetLabel()
end
--是否显示拖动获取
function LTableView:showGetLabel( ... )
    if self.totalCellNums > self.showNums then
        if self.getLabel then
            self.getLabel:removeFromParent()
            self.getLabel = nil
        end
        local cell = self.cells[self.showNums]
        local _size = cell:getContentSize()

        self.getLabel = ui.newTTFLabel({
            text = '获取更多',
            size = 24,
            color = ccc3(247, 62, 49),
        })
        self.cells[self.showNums]:addChild(self.getLabel)
        self.getLabel:setPosition(_size.width/2, - 20)
        self.getLabel:setVisible(true)
    else
        if self.getLabel then
            self.getLabel:removeFromParent()
            self.getLabel = nil
        end
    end
end
--获取增加的cell数量
function LTableView:getAddNums( ... )
    if self.totalCellNums > (self.showNums + 5) then 
        return 5
    else
        return self.totalCellNums - self.showNums
    end
end
--设置cell位置
function LTableView:setCellPos( offset )
    local x, y = offset.x, offset.y
    for i = 1, self.showNums do
        local cell = self.cells[i]
        local cx, cy = cell:getPosition()
        cell:setPosition(cx + offset.x, cy + offset.y)
    end
end
--cell滑动动作
function LTableView:cellMoveAni( offsetY_ani, start )
    if not start then start = 1 end
    for i = start, self.showNums do
        local cell = self.cells[i]
        local cx, cy = cell:getPosition()
        local ac = CCMoveTo:create(0.4, ccp(cx, cy - offsetY_ani))
        cell:runAction(ac)
    end
end
--删除一个Cell
function LTableView:removeOneCell( index )
    self.totalCellNums = self.totalCellNums - 1
    self.showNums = self.showNums - 1
    for i = index, #self.cells - 1 do
        self.cells[index] = self.cells[index + 1]
    end
    self.cells[#self.cells]:removeFromParent()
    self.cells[#self.cells] = nil

    local _size_first = self.cells[1]:getContentSize()
    local offsetY_ani = _size_first.height
    self:cellMoveAni(offsetY_ani, index)
end

function LTableView:getCellWithIndex( ... )
    return self.cells[self.currIndex]
end

--移除所有cell和数据
function LTableView:clean( ... )
    if self.getLabel then
        self.getLabel:removeFromParent()
        self.getLabel = nil
    end
    self.view:removeAllChildrenWithCleanup(true)
    self.cells = {}
    self.view:setPosition(0, 0)
    self.showNums = 0
    self.totalHeight = 0
    self.totalCellNums = 0
end

return LTableView
