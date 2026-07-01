-- Discord-GitHub connected
--[[
	ItemPreviewController
	Swaps the flat icon for a 3D preview on hover.
--]]

local MarketplaceService = game:GetService("MarketplaceService")
local ReplicatedStorage  = game:GetService("ReplicatedStorage")
local RunService         = game:GetService("RunService")
local TweenService       = game:GetService("TweenService")

local Knit    = require(ReplicatedStorage:WaitForChild("Packages"):WaitForChild("Knit"))
local Promise = require(ReplicatedStorage:WaitForChild("Packages"):WaitForChild("Promise"))

local ItemPreviewController = Knit.CreateController { Name = "ItemPreviewController" }

local AssetTypes = require(ReplicatedStorage.Information.AssetTypes)

local Service

local SPINNER_IMAGE = "rbxasset://textures/loading/robloxTilt.png"
local FADE_TIME     = 0.18
local ZOOM_OUT      = 2.8
local FOV_BIAS      = -2

-- cam settings per asset type
local TYPE_TO_PART = {
	Hat = { part="Head", yOff=0.3, z=2.5, fov=22 },
	HairAccessory = { part="Head", yOff=0.2, z=2.5, fov=22 },
	FaceAccessory = { part="Head", yOff=0.0, z=2.0, fov=18 },
	EarAccessory = { part="Head", yOff=0.0, z=2.0, fov=18 },
	EyeAccessory = { part="Head", yOff=0.0, z=2.0, fov=18 },
	EyebrowAccessory = { part="Head", yOff=0.0, z=2.0, fov=18 },
	EyelashAccessory = { part="Head", yOff=0.0, z=2.0, fov=18 },
	Face = { part="Head", yOff=0.0, z=2.0, fov=18 },
	DynamicHead = { part="Head", yOff=0.0, z=2.2, fov=20 },
	Head = { part="Head", yOff=0.0, z=2.2, fov=20 },
	NeckAccessory = { part="UpperTorso", yOff=0.5, z=2.6, fov=24 },
	ShoulderAccessory = { part="UpperTorso", yOff=0.2, z=3.0, fov=28 },
	FrontAccessory = { part="UpperTorso", yOff=0.0, z=3.0, fov=28 },
	BackAccessory = { part="UpperTorso", yOff=0.0, z=3.0, fov=28 },
	WaistAccessory = { part="LowerTorso", yOff=0.0, z=3.0, fov=28 },
	Shirt = { part="UpperTorso", yOff=0.0, z=3.2, fov=30 },
	ShirtAccessory = { part="UpperTorso", yOff=0.0, z=3.2, fov=30 },
	TShirt = { part="UpperTorso", yOff=0.0, z=3.2, fov=30 },
	TShirtAccessory = { part="UpperTorso", yOff=0.0, z=3.2, fov=30 },
	JacketAccessory = { part="UpperTorso", yOff=-0.2, z=3.4, fov=32 },
	SweaterAccessory = { part="UpperTorso", yOff=-0.2, z=3.4, fov=32 },
	Torso = { part="UpperTorso", yOff=-0.1, z=3.4, fov=30 },
	RightArm = { part="RightUpperArm", yOff=0.0, z=3.2, fov=28 },
	LeftArm = { part="LeftUpperArm", yOff=0.0, z=3.2, fov=28 },
	Pants = { part="LeftUpperLeg", yOff=0.0, z=3.4, fov=30 },
	PantsAccessory = { part="LeftUpperLeg", yOff=0.0, z=3.4, fov=30 },
	ShortsAccessory = { part="LowerTorso", yOff=-0.3, z=3.2, fov=28 },
	DressSkirtAccessory = { part="LowerTorso", yOff=-0.3, z=3.4, fov=30 },
	RightLeg = { part="RightUpperLeg", yOff=0.0, z=3.4, fov=30 },
	LeftLeg = { part="LeftUpperLeg", yOff=0.0, z=3.4, fov=30 },
	LeftShoeAccessory = { part="LeftFoot", yOff=0.0, z=2.4, fov=22 },
	RightShoeAccessory = { part="RightFoot", yOff=0.0, z=2.4, fov=22 },
}
-- fallback if type's unknown
local DEFAULT_PART = { part="UpperTorso", yOff=0.0, z=3.5, fov=30 }

-- unknown key just returns DEFAULT_PART, no nil checks needed
setmetatable(TYPE_TO_PART, { __index = function() return DEFAULT_PART end })

local function GetPartCfg(info)
	if not info then return DEFAULT_PART end
	local typeName = AssetTypes[info.AssetTypeId]
	return typeName and TYPE_TO_PART[typeName] or DEFAULT_PART
end

-- clone goes in here, kill anims/movement, aim the cam
local function PlaceInViewport(wm, cam, model, info)
	wm:ClearAllChildren()
	local clone = model:Clone()
	clone.Parent = wm

	-- scripts do nothing in a viewport, yeet them
	for _, d in ipairs(clone:GetDescendants()) do
		if d:IsA("BaseScript") then d:Destroy() end
	end

	-- freeze the rig, no idle anim/walking/tags
	local hum = clone:FindFirstChildOfClass("Humanoid")
	if hum then
		hum.DisplayDistanceType   = Enum.HumanoidDisplayDistanceType.None
		hum.HealthDisplayDistance = 0
		hum.NameDisplayDistance   = 0
		hum.WalkSpeed = 0
		hum.JumpPower = 0
		hum.AutoRotate = false
		local animator = hum:FindFirstChildOfClass("Animator")
		if animator then
			for _, track in ipairs(animator:GetPlayingAnimationTracks()) do track:Stop(0) end
		end
	end

	-- focus part -> HRP -> bounding box, whatever's there
	local cfg       = GetPartCfg(info)
	local focusPart = clone:FindFirstChild(cfg.part)
	local hrp       = clone:FindFirstChild("HumanoidRootPart")
	local centerX   = hrp and hrp.Position.X or 0
	local centerZ   = hrp and hrp.Position.Z or 0
	local centerY
	if focusPart then centerY = focusPart.Position.Y + cfg.yOff
	elseif hrp then centerY = hrp.Position.Y
	else centerY = clone:GetBoundingBox().Position.Y end
	local target = Vector3.new(centerX, centerY, centerZ)

	cam.FieldOfView = math.max(10, cfg.fov + FOV_BIAS)
	cam.CFrame = CFrame.lookAt(target + Vector3.new(0, 0, -cfg.z * ZOOM_OUT), target)
end

-- only one fetch at a time, token = who owns it
local ActiveFetch      = nil
local ActiveFetchToken = nil

local function FetchProductInfoPromise(itemId)
	return Promise.new(function(resolve)
		task.spawn(function()
			local ok, result = pcall(MarketplaceService.GetProductInfo, MarketplaceService, itemId, Enum.InfoType.Asset)
			resolve(ok and result or nil)
		end)
	end)
end

-- gotta wait till Ready flag flips + Model shows up w/ its parts
local function WaitForFolderReady(folder)
	if not folder:GetAttribute("Ready") then
		folder:GetAttributeChangedSignal("Ready"):Wait()
	end
	local m = folder:FindFirstChildOfClass("Model")
	if not m then folder.ChildAdded:Wait(); m = folder:FindFirstChildOfClass("Model") end
	if not m then return nil end
	m:WaitForChild("HumanoidRootPart", 5)
	m:WaitForChild("Humanoid", 5)
	return m
end

-- goes and grabs the model, bails clean if cancelled
local function FetchPreviewModelPromise(itemId)
	return Promise.new(function(resolve, reject, onCancel)
		local cancelled = false
		onCancel(function() cancelled = true end)
		task.spawn(function()
			local folder = Service:GetPreviewModel(itemId)
			if cancelled then return end
			if not folder or not folder.Parent then reject("no preview model") return end
			local model = WaitForFolderReady(folder)
			if cancelled then return end
			if not model then reject("model not ready") return end
			resolve(folder)
		end)
	end)
end

-- got info already? just grab model. if not, fetch both
local function BuildFetchPromise(itemId, knownInfo)
	if knownInfo then
		return FetchPreviewModelPromise(itemId):andThen(function(folder) return { knownInfo, folder } end)
	end
	return Promise.all({ FetchProductInfoPromise(itemId), FetchPreviewModelPromise(itemId) })
end

local function CancelActiveFetch()
	if ActiveFetch and ActiveFetch:getStatus() == Promise.Status.Started then ActiveFetch:cancel() end
	ActiveFetch = nil
	ActiveFetchToken = nil
end

-- main setup, this is what wires everything up for a card
function ItemPreviewController:AttachToButton(button, itemId, productInfo)
	if button:FindFirstChild("ItemPreviewVP") then return end -- already wired
	local icon = button:FindFirstChild("Icon")
	if not icon then return end

	-- the 3D window, sits over the icon, starts hidden
	local vp = Instance.new("ViewportFrame")
	vp.Name                   = "ItemPreviewVP"
	vp.AnchorPoint            = Vector2.new(0.5, 0.5)
	vp.Position               = UDim2.fromScale(0.5, 0.5)
	vp.Size                   = UDim2.fromScale(1, 1)
	vp.BackgroundTransparency = 1
	vp.BorderSizePixel        = 0
	vp.ClipsDescendants       = true
	vp.ZIndex                 = math.max(icon.ZIndex - 1, 0)
	vp.Visible                = false
	vp.Active                 = false
	vp.ImageTransparency      = 1
	vp.Parent                 = button

	-- match the card's corner radius
	do
		local sourceCorner = button:FindFirstChildOfClass("UICorner") or icon:FindFirstChildOfClass("UICorner")
		local uc = Instance.new("UICorner")
		uc.CornerRadius = sourceCorner and sourceCorner.CornerRadius or UDim.new(0, 8)
		uc.Parent = vp
	end

	local wm = Instance.new("WorldModel")
	wm.Parent = vp

	local cam = Instance.new("Camera")
	cam.CameraType   = Enum.CameraType.Scriptable
	cam.FieldOfView  = 30
	cam.Parent       = vp
	vp.CurrentCamera = cam

	-- 2D thumb shown while loading
	local thumb = Instance.new("ImageLabel")
	thumb.Name                   = "Thumbnail"
	thumb.BackgroundTransparency = 1
	thumb.Size                   = UDim2.fromScale(1, 1)
	thumb.Position               = UDim2.fromScale(0, 0)
	thumb.ZIndex                 = vp.ZIndex
	thumb.ScaleType              = Enum.ScaleType.Fit
	if icon:IsA("ImageLabel") or icon:IsA("ImageButton") then thumb.Image = icon.Image end
	thumb.Parent = vp

	-- loading spinner
	local spinner = Instance.new("ImageLabel")
	spinner.Name                   = "Spinner"
	spinner.BackgroundTransparency = 1
	spinner.AnchorPoint            = Vector2.new(0.5, 0.5)
	spinner.Position               = UDim2.fromScale(0.5, 0.5)
	spinner.Size                   = UDim2.fromScale(0.35, 0.35)
	spinner.SizeConstraint         = Enum.SizeConstraint.RelativeXY
	spinner.Image                  = SPINNER_IMAGE
	spinner.ImageTransparency      = 1
	spinner.ZIndex                 = vp.ZIndex + 2
	spinner.Visible                = false
	spinner.Parent                 = vp

	-- per-card state
	local cachedFolder = nil
	local cachedInfo   = productInfo
	local hideThread   = nil
	local hovering     = false
	local spinnerConn  = nil
	local myToken      = nil

	local function startSpinner()
		if spinnerConn then return end
		spinner.Visible  = true
		spinner.Rotation = 0
		TweenService:Create(spinner, TweenInfo.new(0.15), { ImageTransparency = 0 }):Play()
		spinnerConn = RunService.RenderStepped:Connect(function(dt)
			spinner.Rotation = (spinner.Rotation + dt * 360) % 360
		end)
	end

	local function stopSpinner()
		if spinnerConn then spinnerConn:Disconnect(); spinnerConn = nil end
		spinner.Visible           = false
		spinner.ImageTransparency = 1
	end

	-- crossfade thumb -> 3D
	local function fadeInModel()
		TweenService:Create(vp,    TweenInfo.new(FADE_TIME), { ImageTransparency = 0 }):Play()
		TweenService:Create(thumb, TweenInfo.new(FADE_TIME), { ImageTransparency = 1 }):Play()
	end

	local function resetVisuals()
		vp.ImageTransparency    = 1
		thumb.ImageTransparency = 0
		stopSpinner()
		wm:ClearAllChildren()
	end

	local function showVP()
		if hideThread then task.cancel(hideThread); hideThread = nil end
		icon.Visible = false
		vp.Visible   = true
	end

	-- tiny delay so it doesn't flicker on fast mouse moves
	local function hideVP()
		hideThread = task.delay(0.04, function()
			hideThread   = nil
			vp.Visible   = false
			icon.Visible = true
			resetVisuals()
		end)
	end

	local function placeIfStillHovering()
		if not hovering then return end
		if not cachedFolder or not cachedFolder.Parent then return end
		local m = cachedFolder:FindFirstChildOfClass("Model")
		if not m then return end
		PlaceInViewport(wm, cam, m, cachedInfo)
		stopSpinner()
		fadeInModel()
	end

	button.MouseEnter:Connect(function()
		hovering = true
		showVP()

		-- cached? skip straight to showing it
		if cachedFolder and cachedFolder.Parent then placeIfStillHovering() return end

		-- else take over the fetch, spin up loader
		CancelActiveFetch()
		myToken          = {}
		ActiveFetchToken = myToken
		thumb.ImageTransparency = 0
		startSpinner()

		local promise = BuildFetchPromise(itemId, cachedInfo)
		ActiveFetch = promise
		promise:andThen(function(results)
			if ActiveFetchToken ~= myToken then return end -- stale, someone else took over
			local info = results[1]; local folder = results[2]
			if info then cachedInfo = info end
			cachedFolder = folder
			if ActiveFetchToken == myToken then ActiveFetch = nil; ActiveFetchToken = nil end
			placeIfStillHovering()
		end):catch(function(err)
			if ActiveFetchToken == myToken then ActiveFetch = nil; ActiveFetchToken = nil end
			if hovering then stopSpinner() end
			if typeof(err) == "table" and err.kind == Promise.Error.Kind.AlreadyCancelled then return end -- expected
		end)
	end)

	button.MouseLeave:Connect(function()
		hovering = false
		hideVP()
	end)
end

function ItemPreviewController:KnitInit() end

function ItemPreviewController:KnitStart()
	Service = Knit.GetService("ItemPreviewService")
end

return ItemPreviewController
