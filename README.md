if writefile then
    writefile("Shibuya.txt", [[local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer

--====================================================--
--=============== CONFIGURAÇÕES GERAIS ===============--
--====================================================--
local ORIGINAL_ANIMATION_ID = "rbxassetid://108695775669287" -- Animação gatilho

local ASSET_ID = "rbxassetid://124671556536335"
local SCALE = 1
local ANIM_FADE_TIME = 0.15 

----- CONFIGS DA ANIMAÇÃO DE QUEBRA (RODA PRIMEIRO) ---
local S1_ANIMATION_ID = "rbxassetid://114083643054208"
local S1_BREAK_SOUND_ID = "rbxassetid://1358442317"
local S1_LERP_SPEED = 0.8 

local S1_RIGHT_OFFSET = Vector3.new(0, -1, 0)
local S1_RIGHT_ROTATION = Vector3.new(90, 0, 0)

local S1_MIDDLE_OFFSET = Vector3.new(0, -1, 0)
local S1_MIDDLE_ROTATION = Vector3.new(270, 0, 0)

local S1_LEFT_OFFSET = Vector3.new(0, -1, 0)
local S1_LEFT_ROTATION = Vector3.new(270, 0, 0)

----- CONFIGS DA ANIMAÇÃO DE GIRO (RODA DEPOIS) ---
local S2_ANIMATION_ID = "rbxassetid://104169044967547"
local S2_SOUND_ID = "rbxassetid://102378142750719"
local S2_START_TIME = 1.69
local S2_END_TIME = 2.38
local S2_SOUND_TIME = 1.7
local S2_LOOPS = 2

local B1_LEFT_OFFSET = Vector3.new(0, -1, 0)
local B1_LEFT_ROTATION = Vector3.new(270, 0, 0)
local B1_RIGHT_OFFSET = Vector3.new(0, -1, 0)
local B1_RIGHT_ROTATION = Vector3.new(90, 0, 0)
local B1_LERP_SPEED = 0.45 

local B2_LEFT_OFFSET = Vector3.new(0, -1, 0)
local B2_LEFT_ROTATION = Vector3.new(240, -150, 90)
local B2_RIGHT_OFFSET = Vector3.new(0, -1, 0)
local B2_RIGHT_ROTATION = Vector3.new(60, -150, 90)
local B2_LERP_SPEED = 0.9

local FINAL_OFFSET = Vector3.new(0.05, -1.1, -1)
local FINAL_ROTATION = Vector3.new(0, 0, 0)

----- CONFIGS DA ANIMAÇÃO FINAL E ARREMESSO ---
local FINAL_ANIMATION_ID = "rbxassetid://89619191167062"
local SONS_ARREMESSO = {4571259077, 4571259077}
local SONS_CRAVAR = {9113160992, 9113160992}
local BASE_ROTATION = CFrame.Angles(math.rad(0), math.rad(0), math.rad(0))

----- CONFIGS DOS SONS DE M1 / M2 EM OVERRIDE -----
local M1_OLD_ID = "rbxassetid://91019449442779"
local M2_OLD_ID = "rbxassetid://135674501661535"

local M1_NEW_ID = "rbxassetid://220834019"
local M2_NEW_ID = "rbxassetid://7441097182"

--====================================================--
--================ CONTROLE DE ESTADOS ================--
--====================================================--
local character = nil
local humanoid = nil
local animator = nil

local activeConnections = {}
local activeSound = nil
local activeTrack = nil
local sparkTemplate = nil
local BreakVfxOriginal = nil
local cloudModel = nil
local isRunningSequence = false 

-- Identificadores únicos para o override de som
local detectedSounds = {} 
local soundOverrideConnection = nil

-- Declaração antecipada das funções de sequência para evitar erros de escopo
local runBreakingSequence

-- Pré-carrega os assets necessários em segundo plano
task.spawn(function()
	local sucesso, BaseAsset = pcall(function() return game:GetObjects(ASSET_ID)[1] end)
	if sucesso and BaseAsset then
		cloudModel = BaseAsset:Clone()
		for _, v in ipairs(cloudModel:GetDescendants()) do
			if v:IsA("BasePart") then
				v.Anchored = false
				v.CanCollide, v.CanTouch, v.CanQuery, v.Massless = false, false, false, true
				v.Size *= SCALE
			elseif v:IsA("SpecialMesh") then
				v.Scale *= SCALE
			end
		end

		local spark = BaseAsset:FindFirstChild("Spark", true)
		if spark then
			sparkTemplate = spark:Clone()
			sparkTemplate.Parent = nil
		end
		local breakVfx = BaseAsset:FindFirstChild("Break", true)
		if breakVfx then
			BreakVfxOriginal = breakVfx:Clone()
			BreakVfxOriginal.Parent = nil
		end
		BaseAsset:Destroy()
	end
end)

-- Limpeza total de instâncias físicas e lógicas antigas
local function clearAllAssets()
	if getgenv().PlayfulCloudAsset then pcall(function() getgenv().PlayfulCloudAsset:Destroy() end) getgenv().PlayfulCloudAsset = nil end
	if getgenv().IdleCloudLeft then pcall(function() getgenv().IdleCloudLeft:Destroy() end) getgenv().IdleCloudLeft = nil end
	if getgenv().IdleCloudRight then pcall(function() getgenv().IdleCloudRight:Destroy() end) getgenv().IdleCloudRight = nil end
	if getgenv().PlayCloudIntact then pcall(function() getgenv().PlayCloudIntact:Destroy() end) getgenv().PlayCloudIntact = nil end
	if getgenv().PlayCloudBroken then pcall(function() getgenv().PlayCloudBroken:Destroy() end) getgenv().PlayCloudBroken = nil end

	if soundOverrideConnection then
		soundOverrideConnection:Disconnect()
		soundOverrideConnection = nil
	end
	detectedSounds = {}
end

-- Desvanece suavemente antes de deletar
local function fadeOutAndDestroy(asset, duration)
	if not asset then return end
	duration = duration or 0.25
	if getgenv().PlayfulCloudAsset == asset then getgenv().PlayfulCloudAsset = nil end
	if getgenv().PlayCloudIntact == asset then getgenv().PlayCloudIntact = nil end
	if getgenv().PlayCloudBroken == asset then getgenv().PlayCloudBroken = nil end
	
	task.spawn(function()
		local descendants = asset:GetDescendants()
		for _, v in ipairs(descendants) do
			if v:IsA("BasePart") and v.Transparency < 1 then
				TweenService:Create(v, TweenInfo.new(duration, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Transparency = 1}):Play()
			elseif v:IsA("Decal") or v:IsA("Texture") then
				TweenService:Create(v, TweenInfo.new(duration, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Transparency = 1}):Play()
			elseif v:IsA("ParticleEmitter") or v:IsA("Light") then
				v.Enabled = false
			end
		end
		task.wait(duration)
		pcall(function() asset:Destroy() end)
	end)
end

local function scaleModel(model, mult)
	for _, v in ipairs(model:GetDescendants()) do
		if v:IsA("BasePart") then
			v.Size *= mult
		elseif v:IsA("SpecialMesh") then
			v.Scale *= mult
		end
	end
end

local function findPart(obj)
	if obj:IsA("BasePart") then return obj end
	for _, v in ipairs(obj:GetDescendants()) do
		if v:IsA("MeshPart") or v:IsA("BasePart") then return v end
	end
	return nil
end

--====================================================--
--=============== SISTEMA DE MONITORAÇÃO DE M1/M2 =====--
--====================================================--
local function stopSoundOverride()
	if soundOverrideConnection then
		soundOverrideConnection:Disconnect()
		soundOverrideConnection = nil
	end
	detectedSounds = {}
end

local function startSoundOverride()
	if soundOverrideConnection then soundOverrideConnection:Disconnect() end
	detectedSounds = {}

	soundOverrideConnection = RunService.Heartbeat:Connect(function()
		-- TRAVA DE SEGURANÇA: Se o personagem sumir, morrer ou o humanoid sumir, desliga o override imediatamente
		if not character or not character.Parent or not humanoid or humanoid.Health <= 0 then
			stopSoundOverride()
			return
		end

		local hrp = character:FindFirstChild("HumanoidRootPart")
		if not hrp then return end

		for _, instance in ipairs(workspace:GetDescendants()) do
			if instance:IsA("Sound") and instance.IsPlaying then
				local soundId = instance.SoundId
				if soundId == M1_OLD_ID or soundId == M2_OLD_ID then
					local parentPart = instance.Parent
					if parentPart and (parentPart:IsA("BasePart") or parentPart:IsA("Attachment")) then
						local position = parentPart:IsA("Attachment") and parentPart.WorldPosition or parentPart.Position
						local distance = (hrp.Position - position).Magnitude

						if distance <= 9 then
							if not detectedSounds[instance] then
								detectedSounds[instance] = true
								instance.Volume = 0

								local replacementId = (soundId == M1_OLD_ID) and M1_NEW_ID or M2_NEW_ID

								local customSound = Instance.new("Sound")
								customSound.SoundId = replacementId
								customSound.Volume = 1
								customSound.RollOffMaxDistance = 100
								customSound.RollOffMinDistance = 10
								customSound.Parent = parentPart
								customSound:Play()

								customSound.Ended:Connect(function()
									customSound:Destroy()
								end)
								
								instance.Stopped:Connect(function()
									detectedSounds[instance] = nil
								end)
								instance.Ended:Connect(function()
									detectedSounds[instance] = nil
								end)
							end
						end
					end
				end
			end
		end
	end)
end

--====================================================--
--=================== SISTEMA DE ARREMESSO =============--
--====================================================--
local function tocarSom(parent, listaDeSons, tempoInicio)
    if not parent then return end
    local escolhido = listaDeSons[math.random(1, #listaDeSons)]
    local sound = Instance.new("Sound")
    sound.SoundId = "rbxassetid://" .. tostring(escolhido)
    sound.Volume = 1
    sound.RollOffMaxDistance = 150
    sound.RollOffMinDistance = 10
    if tempoInicio then sound.TimePosition = tempoInicio end
    sound.Parent = parent
    sound:Play()
    sound.Ended:Connect(function() sound:Destroy() end)
end

local function prepararProjetil(part)
    part.Anchored = true
    part.CanCollide, part.CanTouch, part.CanQuery = false, false, false
    part.Size = Vector3.new(0.5, 0.5, 2)
    part.Color = Color3.fromRGB(255, 0, 0)
    part.Material = Enum.Material.Neon
    part.Transparency = 1 

    local attachment0 = Instance.new("Attachment")
    attachment0.Name = "TrailAttachment0"
    attachment0.Position = Vector3.new(0, 0.5, 0)
    attachment0.Parent = part

    local attachment1 = Instance.new("Attachment")
    attachment1.Name = "TrailAttachment1"
    attachment1.Position = Vector3.new(0, -0.5, 0)
    attachment1.Parent = part

    local trail = Instance.new("Trail")
    trail.Name = "ProjetilTrail"
    trail.Attachment0 = attachment0
    trail.Attachment1 = attachment1
    trail.Texture = "rbxassetid://9486565084"
    trail.Lifetime = 0.5
    trail.MinLength = 0.1
    trail.WidthScale = NumberSequence.new(1, 0)
    trail.FaceCamera = true 
    trail.Parent = part
end

local function aplicarFadeESumir(projetilPart, visualClone)
    task.wait(6)
    local trail = projetilPart:FindFirstChild("ProjetilTrail")
    if trail then trail.Enabled = false end

    local tweenInfo = TweenInfo.new(1, Enum.EasingStyle.Linear)
    if visualClone then
        for _, part in ipairs(visualClone:GetDescendants()) do
            if part:IsA("BasePart") then TweenService:Create(part, tweenInfo, {Transparency = 1}):Play() end
        end
    else
        TweenService:Create(projetilPart, tweenInfo, {Transparency = 1}):Play()
    end
    task.delay(1, function()
        projetilPart:Destroy()
        if visualClone then visualClone:Destroy() end
    end)
end

local function arremessarProjetil()
    local origPart = character:FindFirstChild("HumanoidRootPart")
    if not origPart then return end

    local projetilPart = Instance.new("Part")
    prepararProjetil(projetilPart)
    projetilPart.Parent = workspace

    local visualClone = nil
    if cloudModel then
        visualClone = cloudModel:Clone()
        visualClone.Parent = workspace
        local visualRoot = findPart(visualClone)
        if visualRoot then
            visualRoot.CFrame = projetilPart.CFrame * BASE_ROTATION
            local weld = Instance.new("WeldConstraint")
            weld.Part0 = projetilPart
            weld.Part1 = visualRoot
            weld.Parent = projetilPart
        end
    else
        projetilPart.Transparency = 0
    end

    tocarSom(projetilPart, SONS_ARREMESSO)

    local lookDirection = origPart.CFrame.LookVector
    local startPos = origPart.Position + (lookDirection * 2)
    projetilPart.CFrame = CFrame.lookAt(startPos, startPos + lookDirection)
    
    local velocity = lookDirection * 110 
    local gravity = Vector3.new(0, -42, 0)
    local timeElapsed = 0
    local cravou = false
    
    local raycastParams = RaycastParams.new()
    raycastParams.FilterDescendantsInstances = {character, projetilPart, visualClone}
    raycastParams.FilterType = Enum.RaycastFilterType.Exclude

    local runConnection
    runConnection = RunService.Heartbeat:Connect(function(dt)
        if cravou then runConnection:Disconnect() return end
        timeElapsed = timeElapsed + dt
        if timeElapsed >= 30 then
            runConnection:Disconnect()
            aplicarFadeESumir(projetilPart, visualClone)
            return
        end
        
        velocity = velocity + (gravity * dt)
        local currentPos = projetilPart.Position
        local nextPos = currentPos + (velocity * dt)
        
        local raycastResult = workspace:Raycast(currentPos, nextPos - currentPos, raycastParams)
        
        if raycastResult then
            cravou = true
            runConnection:Disconnect()
            projetilPart.CFrame = CFrame.lookAt(raycastResult.Position, raycastResult.Position + velocity)
            
            tocarSom(projetilPart, SONS_CRAVAR, 0.6)

            local hitPart = raycastResult.Instance
            if hitPart and not hitPart.Anchored then
                projetilPart.Anchored = false
                local weld = Instance.new("WeldConstraint")
                weld.Part0 = hitPart
                weld.Part1 = projetilPart
                weld.Name = "StickWeld"
                weld.Parent = projetilPart
            else
                projetilPart.Anchored = true
            end
            
            aplicarFadeESumir(projetilPart, visualClone)
        else
            projetilPart.CFrame = CFrame.lookAt(nextPos, nextPos + velocity)
        end
    end)
end

--====================================================--
--================ SESSÃO FINAL =======================--
--====================================================--
local function runFinalThrowSequence()
	local finalAnimation = Instance.new("Animation")
	finalAnimation.AnimationId = FINAL_ANIMATION_ID

	local finalTrack = animator:LoadAnimation(finalAnimation)
	finalTrack.Priority = Enum.AnimationPriority.Action4
	finalTrack:Play(ANIM_FADE_TIME)

	local arremessou = false

	local finalAnimConnection
	finalAnimConnection = RunService.Heartbeat:Connect(function()
		if not finalTrack.IsPlaying then
			finalAnimConnection:Disconnect()
			isRunningSequence = false 
			return
		end

		if not arremessou and finalTrack.TimePosition >= 0.7 then
			arremessou = true
			task.spawn(function()
				if getgenv().IdleCloudRight then
					fadeOutAndDestroy(getgenv().IdleCloudRight, 0.15)
				end
				arremessarProjetil()
			end)
		end
	end)
	table.insert(activeConnections, finalAnimConnection)
end

--====================================================--
--=================== SEQUÊNCIA 2 ====================--
--====================================================--
local function spawnSparkEffect()
	if not sparkTemplate then return end
	local hrp = character:FindFirstChild("HumanoidRootPart")
	if not hrp then return end

	local effectPart = Instance.new("Part")
	effectPart.Size = Vector3.new(1, 1, 1)
	effectPart.Transparency = 1
	effectPart.Anchored = true
	effectPart.CanCollide, effectPart.CanTouch, effectPart.CanQuery, effectPart.Massless = false, false, false, true
	effectPart.CFrame = hrp.CFrame * CFrame.new(0, 4, 0)
	effectPart.Parent = workspace

	local activeEffects = {}
	local function clonarEfeito(item)
		if item:IsA("ParticleEmitter") or item:IsA("Trail") or item:IsA("Beam") or item:IsA("Light") then
			local effectClone = item:Clone()
			effectClone.Parent = effectPart
			if effectClone:IsA("ParticleEmitter") then effectClone.Enabled = true end
			table.insert(activeEffects, effectClone)
		end
	end

	if #sparkTemplate:GetChildren() > 0 or sparkTemplate:IsA("BasePart") or sparkTemplate:IsA("Attachment") then
		for _, item in ipairs(sparkTemplate:GetDescendants()) do clonarEfeito(item) end
	end
	clonarEfeito(sparkTemplate)

	task.wait(0.3)
	for _, effect in ipairs(activeEffects) do
		if effect:IsA("ParticleEmitter") or effect:IsA("Trail") or effect:IsA("Beam") or effect:IsA("Light") then
			effect.Enabled = false
		end
	end
	task.delay(3, function() effectPart:Destroy() end)
end

local function prepareAssetParts(asset)
	local idle = asset:FindFirstChild("PlayfulCloud", true)
	if not idle then return nil end

	local left = idle:FindFirstChild("Left", true)
	local middle = idle:FindFirstChild("Middle", true)
	local right = idle:FindFirstChild("Right", true)

	if middle then
		if middle:IsA("BasePart") then middle.Transparency = 1 end
		for _, child in ipairs(middle:GetDescendants()) do
			if child:IsA("Decal") or child:IsA("Texture") or child:IsA("BasePart") then
				child.Transparency = 1
			elseif child:IsA("ParticleEmitter") or child:IsA("Light") then
				child.Enabled = false
			end
		end
	end

	for _, child in ipairs(idle:GetDescendants()) do
		if child.Name == "C" or child.Name == "C33" or child:IsA("Constraint") or child:IsA("Beam") then
			child:Destroy()
		end
	end

	scaleModel(idle, 1)
	return left, right
end

local function runSpinningSequence()
	local leftHand = character:FindFirstChild("LeftHand") or character:FindFirstChild("Left Arm")
	local rightHand = character:FindFirstChild("RightHand") or character:FindFirstChild("Right Arm")
	if not leftHand or not rightHand then isRunningSequence = false return end

	local success, asset1 = pcall(function() return game:GetObjects(ASSET_ID)[1] end)
	if not success or not asset1 then isRunningSequence = false return end
	
	asset1.Parent = workspace
	getgenv().PlayfulCloudAsset = asset1
	
	local left1, right1 = prepareAssetParts(asset1)
	if not left1 or not right1 then isRunningSequence = false return end

	for _, v in ipairs(asset1:GetDescendants()) do
		if v:IsA("BasePart") then
			v.Anchored = (v == left1 or v == right1)
			v.CanCollide, v.CanTouch, v.CanQuery, v.Massless = false, false, false, true
		end
	end

	local rotL1 = CFrame.fromEulerAnglesXYZ(math.rad(B1_LEFT_ROTATION.X), math.rad(B1_LEFT_ROTATION.Y), math.rad(B1_LEFT_ROTATION.Z))
	local rotR1 = CFrame.fromEulerAnglesXYZ(math.rad(B1_RIGHT_ROTATION.X), math.rad(B1_RIGHT_ROTATION.Y), math.rad(B1_RIGHT_ROTATION.Z))
	left1.CFrame = leftHand.CFrame * CFrame.new(B1_LEFT_OFFSET) * rotL1
	right1.CFrame = rightHand.CFrame * CFrame.new(B1_RIGHT_OFFSET) * rotR1

	local conn1 = RunService.RenderStepped:Connect(function()
		local targetL = (leftHand.CFrame * CFrame.new(B1_LEFT_OFFSET)) * rotL1
		local targetR = (rightHand.CFrame * CFrame.new(B1_RIGHT_OFFSET)) * rotR1
		left1.CFrame = left1.CFrame:Lerp(targetL, B1_LERP_SPEED)
		right1.CFrame = right1.CFrame:Lerp(targetR, B1_LERP_SPEED)
	end)
	table.insert(activeConnections, conn1)

	task.wait(0.5)
	conn1:Disconnect()
	fadeOutAndDestroy(asset1, 0.25)

	local success2, asset2 = pcall(function() return game:GetObjects(ASSET_ID)[1] end)
	if not success2 or not asset2 then isRunningSequence = false return end
	
	asset2.Parent = workspace
	getgenv().PlayCloudBroken = asset2
	
	local left2, right2 = prepareAssetParts(asset2)
	if not left2 or not right2 then isRunningSequence = false return end

	for _, v in ipairs(asset2:GetDescendants()) do
		if v:IsA("BasePart") then
			v.Anchored = (v == left2 or v == right2)
			v.CanCollide, v.CanTouch, v.CanQuery, v.Massless = false, false, false, true
		end
	end

	local rotL2 = CFrame.fromEulerAnglesXYZ(math.rad(B2_LEFT_ROTATION.X), math.rad(B2_LEFT_ROTATION.Y), math.rad(B2_LEFT_ROTATION.Z))
	local rotR2 = CFrame.fromEulerAnglesXYZ(math.rad(B2_RIGHT_ROTATION.X), math.rad(B2_RIGHT_ROTATION.Y), math.rad(B2_RIGHT_ROTATION.Z))
	left2.CFrame = leftHand.CFrame * CFrame.new(B2_LEFT_OFFSET) * rotL2
	right2.CFrame = rightHand.CFrame * CFrame.new(B2_RIGHT_OFFSET) * rotR2

	local conn2 = RunService.RenderStepped:Connect(function()
		local targetL = (leftHand.CFrame * CFrame.new(B2_LEFT_OFFSET)) * rotL2
		local targetR = (rightHand.CFrame * CFrame.new(B2_RIGHT_OFFSET)) * rotR2
		left2.CFrame = left2.CFrame:Lerp(targetL, B2_LERP_SPEED)
		right2.CFrame = right2.CFrame:Lerp(targetR, B2_LERP_SPEED)
	end)
	table.insert(activeConnections, conn2)

	local function spawnAndWeld(handObj, globalKey)
		local success3, finalAsset = pcall(function() return game:GetObjects(ASSET_ID)[1] end)
		if not success3 or not finalAsset then return end

		getgenv()[globalKey] = finalAsset
		finalAsset.Parent = workspace

		local root = findPart(finalAsset)
		if not root then finalAsset:Destroy() return end

		if finalAsset:IsA("Model") then pcall(function() finalAsset.PrimaryPart = root end) end

		for _, v in ipairs(finalAsset:GetDescendants()) do
			if v:IsA("BasePart") then
				v.Anchored = false
				v.CanCollide, v.CanTouch, v.CanQuery, v.Massless = false, false, false, true
				v.Transparency = 1
			end
		end

		root.CFrame = handObj.CFrame * CFrame.new(FINAL_OFFSET) * CFrame.Angles(math.rad(FINAL_ROTATION.X), math.rad(FINAL_ROTATION.Y), math.rad(FINAL_ROTATION.Z))

		for _, x in ipairs(root:GetChildren()) do
			if x:IsA("Weld") or x:IsA("WeldConstraint") or x:IsA("Motor6D") then x:Destroy() end
		end

		local weld = Instance.new("WeldConstraint")
		weld.Part0 = handObj
		weld.Part1 = root
		weld.Parent = root

		for _, v in ipairs(finalAsset:GetDescendants()) do
			if v:IsA("BasePart") then
				TweenService:Create(v, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Transparency = 0}):Play()
			end
		end

		if globalKey == "IdleCloudLeft" then
			task.spawn(startSoundOverride)
			task.delay(999999999, function()
				local targetLeftCloud = getgenv().IdleCloudLeft
				if targetLeftCloud then
					stopSoundOverride()
					fadeOutAndDestroy(targetLeftCloud, 1)
				end
			end)
		end
	end

	local animation = Instance.new("Animation")
	animation.AnimationId = S2_ANIMATION_ID
	activeTrack = animator:LoadAnimation(animation)
	activeTrack.Priority = Enum.AnimationPriority.Action

	activeSound = Instance.new("Sound")
	activeSound.SoundId = S2_SOUND_ID
	activeSound.Volume = 0.4
	activeSound.Parent = character:WaitForChild("HumanoidRootPart")

	local equippedFinal = false 

	for i = 1, S2_LOOPS do
		activeTrack:Play(ANIM_FADE_TIME)
		repeat task.wait() until activeTrack.Length > 0  
		activeTrack.TimePosition = S2_START_TIME  

		local playedSound = false  
		local playedEffect = false  

		while activeTrack.IsPlaying do  
			local t = activeTrack.TimePosition  
			if not playedSound and t >= S2_SOUND_TIME then  
				playedSound = true  
				if activeSound then activeSound:Play() end
			end  
			if not playedEffect and t >= 1.7 then
				playedEffect = true
				task.spawn(spawnSparkEffect)
			end

			if i == 2 and not equippedFinal and t >= (S2_END_TIME - 0.38) then
				equippedFinal = true
				task.spawn(function()
					conn2:Disconnect()
					fadeOutAndDestroy(asset2, 0.15)
					spawnAndWeld(leftHand, "IdleCloudLeft")
					spawnAndWeld(rightHand, "IdleCloudRight")
				end)
			end

			if t >= S2_END_TIME then  
				activeTrack:Stop(ANIM_FADE_TIME)  
				break  
			end  
			task.wait()  
		end  
		if i < S2_LOOPS then task.wait(0.05) end
	end

	task.wait(0.1)
	if not equippedFinal then
		conn2:Disconnect()
		fadeOutAndDestroy(asset2, 0.15)
		spawnAndWeld(leftHand, "IdleCloudLeft")
		spawnAndWeld(rightHand, "IdleCloudRight")
	end

	task.wait(0.3)
	runFinalThrowSequence()
end

--====================================================--
--=================== SEQUÊNCIA 1 ====================--
--====================================================--
runBreakingSequence = function()
	local leftHand = character:FindFirstChild("LeftHand") or character:FindFirstChild("Left Arm")
	local rightHand = character:FindFirstChild("RightHand") or character:FindFirstChild("Right Arm")
	if not leftHand or not rightHand then isRunningSequence = false return end

	local BaseAsset = game:GetObjects(ASSET_ID)[1]
	
	local AssetIntact = BaseAsset:Clone()
	AssetIntact.Parent = workspace
	getgenv().PlayCloudIntact = AssetIntact

	local IdleIntact = AssetIntact:FindFirstChild("PlayfulCloud", true)
	local MiddleIntact = IdleIntact:FindFirstChild("Middle", true)
	local RightIntact = IdleIntact:FindFirstChild("Right", true)

	local LeftPartIntact = IdleIntact:FindFirstChild("Left", true)
	if LeftPartIntact then LeftPartIntact:Destroy() end

	local function ApplyRedColor(instance)
		if instance:IsA("BasePart") then
			instance.Color = Color3.fromRGB(170, 0, 0)
			instance.Transparency = 0
		end
		for _, child in ipairs(instance:GetDescendants()) do
			if child:IsA("BasePart") then
				child.Color = Color3.fromRGB(170, 0, 0)
				child.Transparency = 0
			elseif child:IsA("SpecialMesh") then
				child.TextureId = "" 
				child.VertexColor = Vector3.new(1.5, 0, 0)
			end
		end
	end
	if MiddleIntact then ApplyRedColor(MiddleIntact) end

	for _, v in ipairs(IdleIntact:GetDescendants()) do
		if v:IsA("BasePart") then
			v.Size *= SCALE
			v.CanCollide, v.CanTouch, v.CanQuery, v.Massless = false, false, false, true
		elseif v:IsA("SpecialMesh") then
			v.Scale *= SCALE
		end
	end

	local AssetBroken = BaseAsset:Clone()
	local IdleBroken = AssetBroken:FindFirstChild("PlayfulCloud", true)
	local LeftBroken = IdleBroken:FindFirstChild("Left", true)
	local MiddleBroken = IdleBroken:FindFirstChild("Middle", true)
	local RightBroken = IdleBroken:FindFirstChild("Right", true)

	if MiddleBroken then
		if MiddleBroken:IsA("BasePart") then
			MiddleBroken.Transparency = 1
			MiddleBroken.LocalTransparencyModifier = 1
		end
		for _, child in ipairs(MiddleBroken:GetDescendants()) do
			if child:IsA("Decal") or child:IsA("Texture") or child:IsA("BasePart") then
				child.Transparency = 1
			elseif child:IsA("ParticleEmitter") or child:IsA("Light") then
				child.Enabled = false
			end
		end
	end

	for _, child in ipairs(IdleBroken:GetDescendants()) do
		if child.Name == "C" or child.Name == "C33" or child:IsA("Constraint") or child:IsA("Beam") then
			child:Destroy()
		end
	end

	for _, v in ipairs(IdleBroken:GetDescendants()) do
		if v:IsA("BasePart") then
			v.Size *= SCALE
			v.CanCollide, v.CanTouch, v.CanQuery, v.Massless = false, false, false, true
		elseif v:IsA("SpecialMesh") then
			v.Scale *= SCALE
		end
	end

	BaseAsset:Destroy()

	if RightIntact then RightIntact.Anchored = true end
	if MiddleIntact then MiddleIntact.Anchored = true end
	if LeftBroken then LeftBroken.Anchored = true end
	if RightBroken then RightBroken.Anchored = true end

	local activeWeapon = "intact"

	local s1_RenderConnection
	s1_RenderConnection = RunService.RenderStepped:Connect(function()
		if not leftHand or not rightHand or not character.Parent then
			s1_RenderConnection:Disconnect()
			return
		end

		local rightRotCFrame = CFrame.fromEulerAnglesXYZ(math.rad(S1_RIGHT_ROTATION.X), math.rad(S1_RIGHT_ROTATION.Y), math.rad(S1_RIGHT_ROTATION.Z))

		if activeWeapon == "intact" then
			local middleRotCFrame = CFrame.fromEulerAnglesXYZ(math.rad(S1_MIDDLE_ROTATION.X), math.rad(S1_MIDDLE_ROTATION.Y), math.rad(S1_MIDDLE_ROTATION.Z))
			local targetRight = (rightHand.CFrame * CFrame.new(S1_RIGHT_OFFSET)) * rightRotCFrame
			local targetMiddle = (leftHand.CFrame * CFrame.new(S1_MIDDLE_OFFSET)) * middleRotCFrame

			if RightIntact then RightIntact.CFrame = RightIntact.CFrame:Lerp(targetRight, S1_LERP_SPEED) end
			if MiddleIntact then MiddleIntact.CFrame = MiddleIntact.CFrame:Lerp(targetMiddle, S1_LERP_SPEED) end
			
		elseif activeWeapon == "broken" then
			local leftRotCFrame = CFrame.fromEulerAnglesXYZ(math.rad(S1_LEFT_ROTATION.X), math.rad(S1_LEFT_ROTATION.Y), math.rad(S1_LEFT_ROTATION.Z))
			local targetLeft = (leftHand.CFrame * CFrame.new(S1_LEFT_OFFSET)) * leftRotCFrame
			local targetRight = (rightHand.CFrame * CFrame.new(S1_RIGHT_OFFSET)) * rightRotCFrame

			if LeftBroken then LeftBroken.CFrame = LeftBroken.CFrame:Lerp(targetLeft, S1_LERP_SPEED) end
			if RightBroken then RightBroken.CFrame = RightBroken.CFrame:Lerp(targetRight, S1_LERP_SPEED) end
		end
	end)
	table.insert(activeConnections, s1_RenderConnection)

	local function SwapToBroken()
		if activeWeapon == "broken" then return end
		activeWeapon = "broken"

		if RightIntact and RightBroken then RightBroken.CFrame = RightIntact.CFrame end
		if leftHand and LeftBroken then
			local initLeftRot = CFrame.fromEulerAnglesXYZ(math.rad(S1_LEFT_ROTATION.X), math.rad(S1_LEFT_ROTATION.Y), math.rad(S1_LEFT_ROTATION.Z))
			LeftBroken.CFrame = leftHand.CFrame * CFrame.new(S1_LEFT_OFFSET) * initLeftRot
		end

		AssetBroken.Parent = workspace
		getgenv().PlayCloudBroken = AssetBroken

		if AssetIntact then AssetIntact:Destroy() end
	end

	local function PlayBreakVfx()
		local hrp = character:FindFirstChild("HumanoidRootPart")
		if not hrp or not BreakVfxOriginal then return end

		local effectPart = Instance.new("Part")
		effectPart.Size = Vector3.new(1, 1, 1)
		effectPart.Transparency = 1
		effectPart.Anchored = true
		effectPart.CanCollide, effectPart.CanTouch, effectPart.CanQuery, effectPart.Massless = false, false, false, true
		effectPart.CFrame = hrp.CFrame * CFrame.new(0, 0, -1.5)
		effectPart.Parent = workspace

		local activeEffects = {}
		local function clonarEfeito(item)
			if item:IsA("ParticleEmitter") or item:IsA("Trail") or item:IsA("Beam") or item:IsA("Light") then
				local effectClone = item:Clone()
				effectClone.Parent = effectPart
				if effectClone:IsA("ParticleEmitter") then effectClone.Enabled = true end
				table.insert(activeEffects, effectClone)
			end
		end

		if #BreakVfxOriginal:GetChildren() > 0 or BreakVfxOriginal:IsA("BasePart") or BreakVfxOriginal:IsA("Attachment") then
			for _, item in ipairs(BreakVfxOriginal:GetDescendants()) do clonarEfeito(item) end
		end
		clonarEfeito(BreakVfxOriginal)

		task.delay(0.3, function()
			for _, effect in ipairs(activeEffects) do
				if effect:IsA("ParticleEmitter") or effect:IsA("Trail") or effect:IsA("Beam") or effect:IsA("Light") then
					effect.Enabled = false
				end
			end
		end)
		Debris:AddItem(effectPart, 3)
	end

	task.wait(0.5)

	local s1_animation = Instance.new("Animation")
	s1_animation.AnimationId = S1_ANIMATION_ID

	activeTrack = animator:LoadAnimation(s1_animation)
	activeTrack.Priority = Enum.AnimationPriority.Action4
	activeTrack:Play(ANIM_FADE_TIME)
	activeTrack:AdjustSpeed(0.8)

	local estado = "normal"

	local s1_AnimConnection
	s1_AnimConnection = RunService.Heartbeat:Connect(function()
		if not activeTrack.IsPlaying then
			s1_AnimConnection:Disconnect()
			return
		end
		
		local currentTime = activeTrack.TimePosition

		if estado == "normal" and currentTime >= 0.2 then
			estado = "travando"
			activeTrack:AdjustSpeed(0)
			
			task.spawn(function()
				task.wait(0.2)
				local sound = Instance.new("Sound")
				sound.SoundId = S1_BREAK_SOUND_ID
				sound.Volume = 1
				sound.PlaybackSpeed = 0.8
				sound.Parent = character:FindFirstChild("HumanoidRootPart")
				sound:Play()
				Debris:AddItem(sound, 3)
			end)
			
			task.delay(0.2, function()
				if activeTrack and activeTrack.IsPlaying then
					SwapToBroken()
					PlayBreakVfx()
					activeTrack:AdjustSpeed(0.8)
					estado = "boosted"
				end
			end)
		end
		
		if estado == "boosted" and currentTime >= 0.4 then
			estado = "desacelerando"
		end
		
		if estado == "desacelerando" then
			if currentTime < 0.6 then
				local progress = (currentTime - 0.4) / (0.6 - 0.4)
				local targetSpeed = 0.8 - (progress * (0.8 - 0.7)) 
				activeTrack:AdjustSpeed(targetSpeed)
			else
				activeTrack:Stop(ANIM_FADE_TIME)
				s1_AnimConnection:Disconnect()
				
				s1_RenderConnection:Disconnect()
				fadeOutAndDestroy(AssetBroken, 0.15)
				
				task.wait(0.15)
				runSpinningSequence()
			end
		end
	end)
	table.insert(activeConnections, s1_AnimConnection)
end

--====================================================--
--=== ATIVAÇÃO FORÇADA / SOLUÇÃO MANUAL DA CUTSCENE ===--
--====================================================--
local function forceStartCutscene()
	if isRunningSequence then return end
	isRunningSequence = true
	
	for _, conn in ipairs(activeConnections) do
		if conn then conn:Disconnect() end
	end
	activeConnections = {}
	
	-- Mantém os itens do ambiente antigos limpos e preserva os novos carregados
	if getgenv().PlayCloudIntact then pcall(function() getgenv().PlayCloudIntact:Destroy() end) end
	if getgenv().PlayCloudBroken then pcall(function() getgenv().PlayCloudBroken:Destroy() end) end
	
	task.spawn(runBreakingSequence)
end

--====================================================--
--=============== CONEXÃO DE ANIMAÇÃO GATILHO ==========--
--====================================================--
local function startListening(targetAnimator)
	local trackConnection
	trackConnection = targetAnimator.AnimationPlayed:Connect(function(animationTrack)
		if animationTrack.Animation and animationTrack.Animation.AnimationId == ORIGINAL_ANIMATION_ID then
			animationTrack:Stop(0)
			forceStartCutscene()
		end
	end)
	table.insert(activeConnections, trackConnection)
end

--====================================================--
--================ CONTROLE DE RESPAWN ================--
--====================================================--
local function onCharacterAdded(newCharacter)
	character = newCharacter
	humanoid = character:WaitForChild("Humanoid")
	animator = humanoid:WaitForChild("Animator")
	
	-- Limpa apenas conexões do ciclo de vida anterior para evitar "Memory Leaks"
	for _, conn in ipairs(activeConnections) do
		if conn then conn:Disconnect() end
	end
	activeConnections = {}
	isRunningSequence = false
	
	startListening(animator)
	
	humanoid.Died:Connect(function()
		for _, conn in ipairs(activeConnections) do
			if conn then conn:Disconnect() end
		end
		if activeTrack then pcall(function() activeTrack:Stop(ANIM_FADE_TIME) end) end
		if activeSound then pcall(function() activeSound:Stop() end) end
		clearAllAssets() -- <--- Aqui ele já limpa e desliga o override de som se morrer.
		isRunningSequence = false
	end)
end

if player.Character then
	task.spawn(onCharacterAdded, player.Character)
end

player.CharacterAdded:Connect(onCharacterAdded)
-- Script feito para Executores (Delta) - Rode apenas uma vez!

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local player = Players.LocalPlayer

local ANIM_ID = "rbxassetid://111986372618783"
local originalWalkSpeed = 16

-- Função que aplica toda a lógica no personagem atual
local function aplicarScriptSpawn(character)
	-- 1. Garante que a cabeça do personagem carregou na engine
	local head = character:WaitForChild("Head", 10)
	if not head then return end -- Se por algum motivo bizarro não carregar em 10s, cancela
	
	-- 2. Espera exatamente 0.1 segundos após o carregamento
	task.wait(0.1)
	
	-- 3. Agora sim inicia o restante do script
	local humanoid = character:WaitForChild("Humanoid", 10)
	local animator = humanoid:WaitForChild("Animator", 10)
	
	if not humanoid or not animator then return end
	
	-- Criando e carregando a animação
	local animation = Instance.new("Animation")
	animation.AnimationId = ANIM_ID
	
	local track = animator:LoadAnimation(animation)
	track.Priority = Enum.AnimationPriority.Action4 -- Prioridade máxima
	
	local animStarted = false
	local interrupted = false
	local spawnTime = os.clock()
	
	-- Trava o WalkSpeed fisicamente na engine para o jogo não burlar
	local speedBinder
	speedBinder = RunService.PreSimulation:Connect(function()
		if not interrupted and character.Parent then
			humanoid.WalkSpeed = 0
			humanoid.JumpPower = 0 -- Impede pulo também
		else
			if speedBinder then speedBinder:Disconnect() end
		end
	end)
	
	-- LOOP DE SPAM (Roda até a animação iniciar)
	task.spawn(function()
		while not animStarted and character.Parent do
			if not track.IsPlaying then
				track:Play(0.1)
			end
			task.wait(0.05)
		end
	end)
	
	-- DETECTOR DE INÍCIO
	task.spawn(function()
		while not track.IsPlaying and character.Parent do
			task.wait()
		end
		animStarted = true
	end)
	
	-- DETECTOR DE INTERRUPÇÃO (Bloqueado por 1 segundo após o spawn)
	local animatorConnection
	animatorConnection = animator.AnimationPlayed:Connect(function(playedTrack)
		local timeSinceSpawn = os.clock() - spawnTime
		
		-- Só aceita interrupção após 1 segundo do spawn
		if timeSinceSpawn >= 1 then
			if playedTrack.Animation.AnimationId ~= ANIM_ID then
				-- Verifica se a nova animação é de prioridade Action (habilidade/ataque)
				if playedTrack.Priority == Enum.AnimationPriority.Action 
					or playedTrack.Priority == Enum.AnimationPriority.Action2 
					or playedTrack.Priority == Enum.Priority.Action3 
					or playedTrack.Priority == Enum.AnimationPriority.Action4 then
					
					interrupted = true
					track:Stop(0.3) -- Para a animação de spawn suavemente
					
					if speedBinder then speedBinder:Disconnect() end
					humanoid.WalkSpeed = originalWalkSpeed
					humanoid.JumpPower = 50
					
					if animatorConnection then animatorConnection:Disconnect() end
				end
			end
		end
	end)
	
	-- FINALIZAÇÃO NATURAL
	track.Stopped:Connect(function()
		if animatorConnection then animatorConnection:Disconnect() end
		if speedBinder then speedBinder:Disconnect() end
		
		if not interrupted then
			humanoid.WalkSpeed = originalWalkSpeed
			humanoid.JumpPower = 50
		end
	end)
end

-- Se você injetar o script e já estiver vivo, ele também valida a cabeça e aplica
if player.Character then
	task.spawn(aplicarScriptSpawn, player.Character)
end

-- Fica ouvindo o jogo para aplicar toda vez que você morrer e der Respawn
player.CharacterAdded:Connect(function(newCharacter)
	aplicarScriptSpawn(newCharacter)
end)]])
    print("Salvou!")
else
    warn("writefile não existe")
end
--==========================================================================================
-- SCRIPT TOTALMENTE UNIFICADO (FUSÃO DOS 12 MÓDULOS SEM ENCURTAMENTO)
--==========================================================================================

local Players = game:GetService("Players")
local Debris = game:GetService("Debris")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Lighting = game:GetService("Lighting")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()

--==========================================================================================
-- CONFIGURAÇÕES GERAIS E DECLARAÇÃO DE VARIÁVEIS DE CADA MÓDULO
--==========================================================================================

-- [Módulo 1: Sound Replacement]
local SWEEFT_LIST = {
	"rbxassetid://120401800831737",
	"rbxassetid://97228090794267",
	"rbxassetid://116901112122156",
}
local OLD_SWEEFT = "4571259077"

-- [Módulo 2: Domain Expansion]
local COOLDOWN_TIME = 10
local cooldownAtivo = false
local tempoRestante = 0
local CUSTOM_ANIM_ID = "rbxassetid://128880305726742"
local targetFolder = ReplicatedStorage:WaitForChild("Modules"):WaitForChild("MVP"):WaitForChild("Domain Invasion")

-- [Módulo 3: Delta Aura]
local DELTA_AURA_ASSET_ID = 105860645419737
local sourceAura = nil
local partMapping = {
	["AuraHead"] = "Head",
	["AuraTorso"] = "Torso",
	["AuraLeftArm"] = "Left Arm",
	["AuraRightArm"] = "Right Arm",
	["AuraLeftLeg"] = "Left Leg",
	["AuraRightLeg"] = "Right Leg"
}
local PART_SETTINGS = {
	AuraHead = { Scale = 1.1, Offset = Vector3.new(0, -0.24, 0.2) },
	AuraTorso = { Scale = 0.01, Offset = Vector3.new(0, 0, 0) },
	AuraLeftArm = { Scale = 0.01, Offset = Vector3.new(0, 0.42, 0) },
	AuraRightArm = { Scale = 0.01, Offset = Vector3.new(0, 0.42, 0) },
	AuraLeftLeg = { Scale = 1.00, Offset = Vector3.new(0, 0, 0) },
	AuraRightLeg = { Scale = 1.00, Offset = Vector3.new(0, 0, 0) }
}

-- [Módulo 4: Skin Changer]
local SHIRT_ID = "http://www.roblox.com/asset/?id=9247932409"
local PANTS_ID = "http://www.roblox.com/asset/?id=16416497086"
local SKIN_COLOR = Color3.fromRGB(250, 229, 215)
local MESH_CONFIGS = {
	{Parte = "Torso", Mesh = "rbxassetid://134134771482815", Texture = "rbxassetid://106167745650443", Offset = Vector3.new(0, 0, 0), Rot = Vector3.new(0, 0, 0)},
	{Parte = "Head", Mesh = "rbxassetid://90796533693815", Texture = "rbxassetid://98136638456627", Offset = Vector3.new(0, 0, 0), Rot = Vector3.new(0, 180, 0)},
	{Parte = "Head", Mesh = "rbxassetid://118910642913606", Texture = "rbxassetid://76029075621469", Offset = Vector3.new(0, 0, 0), Rot = Vector3.new(0, 180, 0)},
	{Parte = "Head", Mesh = "rbxassetid://97609265699073", Texture = "rbxassetid://92792453689364", Offset = Vector3.new(0.05, 0.2, 0.1), Rot = Vector3.new(0, 0, 0)},
	{Parte = "Right Arm", Mesh = "rbxassetid://99328692547707", Texture = "rbxassetid://112782876026126", Offset = Vector3.new(0, 0.3, 0), Rot = Vector3.new(0, 0, 0)},
	{Parte = "Left Arm", Mesh = "rbxassetid://105256213077864", Texture = "rbxassetid://112782876026126", Offset = Vector3.new(0, 0.3, 0), Rot = Vector3.new(0, 0, 0)}
}

-- [Módulo 5: Teleport Cargas]
local MAX_CHARGES = 5
local currentCharges = MAX_CHARGES
local COOLDOWN_TP = 0.8
local cooldownTpAtivo = false
local tpEffectModel = nil
local CoresPorCarga = {
    [5] = Color3.fromRGB(34, 139, 34),
    [4] = Color3.fromRGB(154, 205, 50),
    [3] = Color3.fromRGB(255, 140, 0),
    [2] = Color3.fromRGB(200, 30, 0),
    [1] = Color3.fromRGB(139, 0, 0),
    [0] = Color3.fromRGB(100, 0, 0)
}

-- [Módulo 6: TP Walk & Ghost]
local TP_SPEED = 0.5
local BOOST_TIME = 0.1
local TP_INTERVAL = 0.9
local GHOST_INTERVAL = 1
local GHOST_COLOR = Color3.fromRGB(30, 30, 30)
local lastTP = os.clock()
local lastGhost = os.clock()
local ghostFolder = workspace:FindFirstChild("BodyTrails") or Instance.new("Folder")
ghostFolder.Name = "BodyTrails"
ghostFolder.Parent = workspace

-- [Módulo 7: Idle Animation]
local idleAnimId = "rbxassetid://89962806326079"
local idleAnimation = Instance.new("Animation")
idleAnimation.AnimationId = idleAnimId

-- [Módulo 8: Hit Effects]
local HIT_ASSET_ID = "rbxassetid://124671556536335"
local HIT_EFFECT_NAME = "Hit"
local HIT_TARGET_ANIMS = {
	["79436586236026"] = true,
	["104137631480391"] = true,
	["102285403332509"] = true,
	["84359513001979"] = true
}

-- [Módulo 9: Barrage 2 Effect]
local BARRAGE2_EFFECT_NAME = "Barrage2"
local BARRAGE2_TARGET_ANIM_ID = "rbxassetid://100811576955331"
local efeitoAtivoBarrage2 = nil

-- [Módulo 10: Barrage 1 Effect]
local BARRAGE1_EFFECT_NAME = "Barrage1"
local BARRAGE1_MAX_DISTANCE = 8
local BARRAGE1_TARGET_SOUNDS = {
	["135674501661535"] = true,
	["139826126503063"] = true,
	["91019449442779"] = true,
	["91344226850535"] = true,
	["78963760295110"] = true,
	["112016169570826"] = true,
	["132547948177910"] = true
}
local conexoesAtivasSons = {}

-- [Módulo 11: Playful Cloud Armamento]
local PLAYFUL_ASSET_ID = "rbxassetid://124671556536335"
local PLAYFUL_SCALE = 1
local PLAYFUL_ATTACK_ANIMS = {
	["84359513001979"] = true,
	["79436586236026"] = true,
	["102285403332509"] = true,
	["104137631480391"] = true,
	["100811576955331"] = true,
	["81210313723714"] = true,
	["81708642912019"] = true,
}
local BLOCK_ANIM_ID = "99032134144831"
local ESTABILIZAR_ANIM_ID = "113359849246757"
local BACK_TO_IDLE_DELAY = 0.8 
local FOLLOW_SPEED = 0.8 
local LERP_SPEED = 0.45 

local IDLE_CONFIG = {
	LeftOffset = Vector3.new(-0.5, -1, 0), LeftRotation = Vector3.new(90, 0, 0),
	RightOffset = Vector3.new(0.5, -1, 0), RightRotation = Vector3.new(90, 0, 0),
	MiddleOffset = Vector3.new(0, -1, 0), MiddleRotation = Vector3.new(270, 0, -10),
}
local ATTACK_CONFIG = {
	LeftOffset = Vector3.new(0, -1, 0), LeftRotation = Vector3.new(90, 0, 0),
	RightOffset = Vector3.new(0, -1, 0), RightRotation = Vector3.new(270, 0, 0),
	MiddleOffset = Vector3.new(1, 0.24, 0), MiddleRotation = Vector3.new(270, 0, 0),
}
local BLOCK_CONFIG = {
	LeftOffset = Vector3.new(0, -1, 0), LeftRotation = Vector3.new(270, 0, -8),
	RightOffset = Vector3.new(0, -1, 0), RightRotation = Vector3.new(90, 0, 70),
	MiddleOffset = Vector3.new(-0.2, 0, 0), MiddleRotation = Vector3.new(270, 0, 0),
}
local ESTABILIZAR_CONFIG = {
	RightOffset = Vector3.new(0, -1, 0), RightRotation = Vector3.new(90, 0, 70),
}

if getgenv().PlayfulCloudAsset then pcall(function() getgenv().PlayfulCloudAsset:Destroy() end) end
local PlayfulAsset = game:GetObjects(PLAYFUL_ASSET_ID)[1]
PlayfulAsset.Parent = workspace
getgenv().PlayfulCloudAsset = PlayfulAsset

local PlayfulIdleModel = PlayfulAsset:FindFirstChild("PlayfulCloud", true)
local PlayfulLeft = PlayfulIdleModel:FindFirstChild("Left", true)
local PlayfulMiddle = PlayfulIdleModel:FindFirstChild("Middle", true)
local PlayfulRight = PlayfulIdleModel:FindFirstChild("Right", true)

local function ScalePlayfulModel(model, mult)
	for _,v in ipairs(model:GetDescendants()) do
		if v:IsA("BasePart") then v.Size *= mult elseif v:IsA("SpecialMesh") then v.Scale *= mult end
	end
end
ScalePlayfulModel(PlayfulIdleModel, PLAYFUL_SCALE)
local originalMiddleSize = PlayfulMiddle.Size

local PlayfulUpdateConnection
local lastActiveAttackTime = 0 

--==========================================================================================
-- ASYNCHRONOUS ASSET LOADING (CARREGAMENTOS EXTERNOS)
--==========================================================================================

-- Download da Aura Delta
task.spawn(function()
	local success, objects = pcall(function()
		return game:GetObjects("rbxassetid://" .. DELTA_AURA_ASSET_ID)
	end)
	if success and objects and #objects > 0 then
		sourceAura = objects[1]
		print("Delta Aura carregada com sucesso.")
	else
		warn("Delta falhou ao baixar o modelo da Aura.")
	end
end)

-- Download do Efeito de Teleporte
task.spawn(function()
    pcall(function()
        local objects = game:GetObjects("rbxassetid://105860645419737")
        if objects and #objects > 0 then
            tpEffectModel = objects[1]
            tpEffectModel.Parent = ReplicatedStorage
        end
    end)
end)

--==========================================================================================
-- FUNÇÕES DE SUPORTE (MÓDULO POR MÓDULO)
--==========================================================================================

-- [Módulo 1: Sound Replacement]
local function playReplacement(parent, soundId, volume, speed)
	local s = Instance.new("Sound")
	s.SoundId = soundId
	s.Volume = volume
	s.PlaybackSpeed = speed
	s.Parent = parent
	s:Play()
	Debris:AddItem(s, 5)
end

local function hookSound(sound)
	if not sound:IsA("Sound") then
		return
	end

	sound.Played:Connect(function()
		local id = tostring(sound.SoundId):match("%d+")
		if not id then
			return
		end

		if id == OLD_SWEEFT then
			playReplacement(
				sound.Parent,
				SWEEFT_LIST[math.random(#SWEEFT_LIST)],
				sound.Volume * 0.25,
				sound.PlaybackSpeed
			)
		end
	end)
end

-- [Módulo 3: Delta Aura Helpers]
local function scaleNumberSequence(ns, mult)
	local keypoints = {}
	for _, kp in ipairs(ns.Keypoints) do
		table.insert(keypoints, NumberSequenceKeypoint.new(
			kp.Time,
			kp.Value * mult,
			kp.Envelope * mult
		))
	end
	return NumberSequence.new(keypoints)
end

local function scaleAuraEffects(obj, mult)
	for _, d in ipairs(obj:GetDescendants()) do
		if d:IsA("Attachment") then
			d.Position *= mult
		elseif d:IsA("ParticleEmitter") then
			pcall(function()
				d.Size = scaleNumberSequence(d.Size, mult)
			end)
		elseif d:IsA("Trail") then
			pcall(function()
				d.WidthScale = scaleNumberSequence(d.WidthScale, mult)
			end)
		elseif d:IsA("Beam") then
			pcall(function()
				d.Width0 *= mult
				d.Width1 *= mult
			end)
		elseif d:IsA("SpecialMesh") then
			pcall(function()
				d.Scale *= mult
			end)
		end
	end
end

local function applyAura(charToApply)
	local oldAura = charToApply:FindFirstChild("DeltaCharacterAura")
	if oldAura then
		oldAura:Destroy()
	end

	charToApply:WaitForChild("HumanoidRootPart")

	local auraFolder = Instance.new("Folder")
	auraFolder.Name = "DeltaCharacterAura"
	auraFolder.Parent = charToApply

	if not sourceAura then
		-- Pequena verificação de atraso caso o download assíncrono ainda não tenha terminado
		local limit = 0
		while not sourceAura and limit < 50 do
			task.wait(0.1)
			limit = limit + 1
		end
	end

	if not sourceAura then return end

	for auraPartName, bodyPartName in pairs(partMapping) do
		local originalPart = sourceAura:FindFirstChild(auraPartName)
		local targetBodyPart = charToApply:FindFirstChild(bodyPartName)

		if originalPart and targetBodyPart and originalPart:IsA("BasePart") then
			local auraClone = originalPart:Clone()
			auraClone.Name = auraPartName
			auraClone.Transparency = 1
			auraClone.CanCollide = false
			auraClone.CanTouch = false
			auraClone.CanQuery = false
			auraClone.Massless = true
			auraClone.Anchored = false

			local settings = PART_SETTINGS[auraPartName]
			if settings then
				if settings.Scale ~= 1 then
					auraClone.Size *= settings.Scale
					scaleAuraEffects(auraClone, settings.Scale)
				end
			end

			auraClone.Parent = auraFolder

			local weld = Instance.new("Weld")
			weld.Part0 = targetBodyPart
			weld.Part1 = auraClone

			if settings then
				weld.C0 = CFrame.new(settings.Offset)
			else
				weld.C0 = CFrame.new()
			end
			weld.Parent = auraClone
		end
	end
end

-- [Módulo 4: Skin Changer Base]
local function applySkin(charToApply)
	for _, obj in ipairs(charToApply:GetChildren()) do
		if obj:IsA("Accessory") or obj:IsA("ShirtGraphic") or obj:IsA("BodyColors") or obj:IsA("CharacterMesh") then
			obj:Destroy()
		end
	end

	for _, part in ipairs(charToApply:GetDescendants()) do
		if part.Name == "SkinCustomMeshPart" then
			part:Destroy()
		end
	end

	for _, part in ipairs(charToApply:GetDescendants()) do
		if part:IsA("BasePart") and part.Name ~= "ManualEffectPart" then
			part.Color = SKIN_COLOR
			part.Material = Enum.Material.SmoothPlastic
			part.Reflectance = 0
		end
	end

	local shirt = charToApply:FindFirstChildOfClass("Shirt") or Instance.new("Shirt", charToApply)
	shirt.ShirtTemplate = SHIRT_ID

	local pants = charToApply:FindFirstChildOfClass("Pants") or Instance.new("Pants", charToApply)
	pants.PantsTemplate = PANTS_ID

	for _, config in ipairs(MESH_CONFIGS) do
		local bodyPart = charToApply:FindFirstChild(config.Parte)
		if bodyPart then
			local meshPart = Instance.new("MeshPart")
			meshPart.Name = "SkinCustomMeshPart"
			meshPart.Size = Vector3.new(1, 1, 1)
			meshPart.CanCollide = false
			meshPart.Massless = true
			meshPart.Anchored = false
			meshPart.MeshId = config.Mesh
			meshPart.TextureID = config.Texture
			
			meshPart.Reflectance = 0 
			meshPart.Material = Enum.Material.SmoothPlastic

			local baseCFrame = bodyPart.CFrame * CFrame.new(config.Offset)
			local rotationCFrame = CFrame.fromEulerAnglesXYZ(math.rad(config.Rot.X), math.rad(config.Rot.Y), math.rad(config.Rot.Z))
			meshPart.CFrame = baseCFrame * rotationCFrame
			meshPart.Parent = bodyPart

			local weld = Instance.new("WeldConstraint")
			weld.Part0 = meshPart
			weld.Part1 = bodyPart
			weld.Parent = meshPart
		end
	end
end

-- [Módulo 5: Teleport Efeitos e Interface]
local function suavizarCorTip(tipLabel, totalCargas)
    local corAlvo = CoresPorCarga[totalCargas] or Color3.fromRGB(255, 255, 255)
    local infoTween = TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
    local animacaoCor = TweenService:Create(tipLabel, infoTween, {TextColor3 = corAlvo})
    animacaoCor:Play()
end

local function aplicarEfeitosTp(torso)
    local sound = Instance.new("Sound")
    sound.SoundId = "rbxassetid://18440678071"
    sound.Volume = 2
    sound.Parent = torso
    
    task.delay(0.05, function()
        if sound and sound.Parent then
            sound:Play()
        end
    end)
    Debris:AddItem(sound, 3)

    if not tpEffectModel then return end
    
    local originalSmokePart = tpEffectModel:FindFirstChild("FloorSmoke") or tpEffectModel:FindFirstChildWhichIsA("BasePart", true)
    if not originalSmokePart then return end
    
    local clonedPart = originalSmokePart:Clone()
    clonedPart.CFrame = torso.CFrame
    
    local weld = Instance.new("WeldConstraint")
    weld.Part0 = torso
    weld.Part1 = clonedPart
    weld.Parent = clonedPart
    clonedPart.Parent = character
    
    clonedPart.Transparency = 1
    clonedPart.CanCollide = false
    clonedPart.Anchored = false

    task.delay(0.5, function()
        if clonedPart then
            for _, fx in pairs(clonedPart:GetDescendants()) do
                if fx:IsA("ParticleEmitter") then
                    fx.Enabled = false
                end
            end
            Debris:AddItem(clonedPart, 4)
        end
    end)
end

local function dispararTeleporteFrente()
    if cooldownTpAtivo or currentCharges <= 0 then return end

    if character:FindFirstChildOfClass("Humanoid") then
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        local animator = humanoid:FindFirstChildOfClass("Animator") or Instance.new("Animator", humanoid)
        
        if animator then
            local animObj = Instance.new("Animation")
            animObj.AnimationId = "rbxassetid://122607727974119"
            
            local track = animator:LoadAnimation(animObj)
            track.Priority = Enum.AnimationPriority.Core
            track:Play()
        end
    end

    local torso = character:FindFirstChild("UpperTorso") or character:FindFirstChild("Torso")
    local root = character:FindFirstChild("HumanoidRootPart")

    if root and torso then
        cooldownTpAtivo = true
        currentCharges = currentCharges - 1
        
        character:PivotTo(character:GetPivot() * CFrame.new(0, 0, -20))
        task.spawn(aplicarEfeitosTp, torso)

        task.delay(COOLDOWN_TP, function()
            cooldownTpAtivo = false
        end)
        
        task.delay(15, function()
            if currentCharges < MAX_CHARGES then
                currentCharges = currentCharges + 1
            end
        end)
    end
end

local function forceInjectButton()
    local playerGui = player:WaitForChild("PlayerGui", 10)
    local movesetFolder = nil
    
    for _, v in pairs(playerGui:GetDescendants()) do
        if v:IsA("GuiObject") and v.Name == "Cleaving Whirlwind" then
            movesetFolder = v.Parent
            break
        end
    end

    if movesetFolder then
        if movesetFolder:FindFirstChild("TeleportSkill") then
            movesetFolder.TeleportSkill:Destroy()
        end

        local sample = movesetFolder:FindFirstChild("Cleaving Whirlwind")
        if sample then
            local tpFrame = sample:Clone()
            tpFrame.Name = "TeleportSkill"
            tpFrame.LayoutOrder = -1 
            
            local originalSize = tpFrame.Size
            tpFrame.Size = UDim2.new(
                originalSize.X.Scale * 0.8, 
                originalSize.X.Offset * 0.8, 
                originalSize.Y.Scale * 0.8, 
                originalSize.Y.Offset * 0.8
            )
            
            if tpFrame:FindFirstChild("Item") then tpFrame.Item:Destroy() end
            if tpFrame:FindFirstChild("Cooldown") then tpFrame.Cooldown:Destroy() end
            
            local tipLabel = tpFrame:FindFirstChild("Tip")
            if tipLabel then
                tipLabel.Visible = true
                tipLabel.TextColor3 = CoresPorCarga[MAX_CHARGES]
            end
            
            local btnOriginal = tpFrame:FindFirstChild("ItemName")
            local keyContainer = tpFrame:FindFirstChild("Key")
            local labelKey = keyContainer and keyContainer:FindFirstChild("Key")
            
            if btnOriginal and btnOriginal:IsA("TextButton") then
                local cloneProps = {
                    Size = btnOriginal.Size,
                    Position = btnOriginal.Position,
                    Font = btnOriginal.Font,
                    TextSize = btnOriginal.TextSize
                }
                btnOriginal:Destroy()

                local newBtn = Instance.new("TextButton")
                newBtn.Name = "ItemName"
                newBtn.Size = cloneProps.Size
                newBtn.Position = cloneProps.Position
                newBtn.Font = cloneProps.Font
                newBtn.TextSize = cloneProps.TextSize
                newBtn.BackgroundTransparency = 1 
                newBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
                newBtn.TextWrapped = true
                newBtn.Parent = tpFrame
                
                newBtn.MouseButton1Click:Connect(dispararTeleporteFrente)
                btnOriginal = newBtn
            end
            
            if labelKey then
                labelKey.Text = "X"
            end

            tpFrame.Parent = movesetFolder

            local ultimaCargaRegistrada = currentCharges

            RunService.RenderStepped:Connect(function()
                if tpFrame and tpFrame.Parent then
                    if btnOriginal and btnOriginal.Text ~= "Flash Step" then btnOriginal.Text = "Flash Step" end
                    if labelKey and labelKey.Text ~= "X" then labelKey.Text = "X" end
                    
                    if tipLabel then
                        tipLabel.Text = "(Step: " .. tostring(currentCharges) .. ")"
                        
                        if currentCharges ~= ultimaCargaRegistrada then
                            suavizarCorTip(tipLabel, currentCharges)
                            ultimaCargaRegistrada = currentCharges
                        end
                    end
                end
            end)
        end
    end
end

-- [Módulo 6: Ghost Trail Creator]
local function CreateGhost()
	local char = player.Character
	if not char then return end

	for _, v in ipairs(char:GetChildren()) do
		if v:IsA("BasePart")
		and v.Name ~= "HumanoidRootPart"
		and not v:FindFirstAncestorWhichIsA("Accessory") then

			local ghost = Instance.new("Part")

			if v.Name == "Head" then
				ghost.Size = v.Size * 1.15
				local mesh = Instance.new("SpecialMesh")
				mesh.MeshType = Enum.MeshType.Head
				mesh.Scale = Vector3.new(1.15, 1.15, 1.15)
				mesh.Parent = ghost
			else
				ghost.Size = v.Size
				ghost.Shape = Enum.PartType.Block
			end

			ghost.CFrame = v.CFrame
			ghost.Anchored = true
			ghost.CanCollide = false
			ghost.CanTouch = false
			ghost.CanQuery = false
			ghost.CastShadow = false

			ghost.Material = Enum.Material.Neon
			ghost.Color = GHOST_COLOR
			ghost.Transparency = (v.Name == "Head") and 0.65 or 0.8
			ghost.Parent = ghostFolder

			task.spawn(function()
				task.wait(2)
				if ghost.Parent then
					local tween = TweenService:Create(
						ghost,
						TweenInfo.new(0.5, Enum.EasingStyle.Linear),
						{Transparency = 1}
					)
					tween:Play()
					tween.Completed:Wait()
					if ghost.Parent then
						ghost:Destroy()
					end
				end
			end)
		end
	end
end

-- [Módulo 7: Idle Animation Core]
local function aplicarAnimacao(charToApply)
    local humanoid = charToApply:WaitForChild("Humanoid", 10)
    if not humanoid then return end
    
    local animator = humanoid:WaitForChild("Animator", 10)
    if not animator then return end

    local track = animator:LoadAnimation(idleAnimation)
    track.Priority = Enum.AnimationPriority.Core 
    track.Looped = true
    
    task.wait(0.5)
    track:Play()
end

-- [Módulo 8: Hit Visual Creator]
local function criarEfeitoHit(meuHrp)
	local sucesso, AssetHitObj = pcall(function() return game:GetObjects(HIT_ASSET_ID)[1] end)
	if not sucesso or not AssetHitObj then return end

	local targetEffect = AssetHitObj:FindFirstChild(HIT_EFFECT_NAME, true)
	if not targetEffect then
		AssetHitObj:Destroy()
		return
	end

	local effectPart = Instance.new("Part")
	effectPart.Size = Vector3.new(1, 1, 1)
	effectPart.Transparency = 1
	effectPart.Anchored = true
	effectPart.CanCollide = false
	effectPart.CanTouch = false
	effectPart.CanQuery = false
	effectPart.Massless = true
	effectPart.CFrame = meuHrp.CFrame * CFrame.new(0, 0, -4)
	effectPart.Parent = workspace

	local activeEffects = {}

	local function clonarEfeito(item)
		if item:IsA("ParticleEmitter") or item:IsA("Trail") or item:IsA("Beam") or item:IsA("Light") then
			local effectClone = item:Clone()
			effectClone.Parent = effectPart
			if effectClone:IsA("ParticleEmitter") then
				effectClone.Enabled = true
			end
			table.insert(activeEffects, effectClone)
		end
	end

	if #targetEffect:GetChildren() > 0 or targetEffect:IsA("BasePart") or targetEffect:IsA("Attachment") then
		for _, item in ipairs(targetEffect:GetDescendants()) do
			clonarEfeito(item)
		end
	end
	clonarEfeito(targetEffect)
	AssetHitObj:Destroy()

	task.wait(0.3)

	for _, effect in ipairs(activeEffects) do
		if effect:IsA("ParticleEmitter") then
			effect.Enabled = false
		elseif effect:IsA("Trail") or effect:IsA("Beam") or effect:IsA("Light") then
			effect.Enabled = false
		end
	end

	task.delay(3, function()
		if effectPart then
			effectPart:Destroy()
		end
	end)
end

-- [Módulo 9: Barrage 2 Handler]
local function iniciarEfeitoNoPlayer(meuHrp)
	if efeitoAtivoBarrage2 then
		efeitoAtivoBarrage2:Destroy()
		efeitoAtivoBarrage2 = nil
	end

	local sucesso, AssetBarrageObj = pcall(function() return game:GetObjects(HIT_ASSET_ID)[1] end)
	if not sucesso or not AssetBarrageObj then return nil end

	local targetEffect = AssetBarrageObj:FindFirstChild(BARRAGE2_EFFECT_NAME, true)
	if not targetEffect then
		AssetBarrageObj:Destroy()
		return nil
	end

	local effectPart = Instance.new("Part")
	effectPart.Size = Vector3.new(1, 1, 1)
	effectPart.Transparency = 1
	effectPart.CanCollide = false
	effectPart.CanTouch = false
	effectPart.CanQuery = false
	effectPart.Massless = true
	effectPart.CFrame = meuHrp.CFrame
	effectPart.Parent = workspace

	local weld = Instance.new("WeldConstraint")
	weld.Part0 = effectPart
	weld.Part1 = meuHrp
	weld.Parent = effectPart

	effectPart.Anchored = false

	local activeEffects = {}

	local function clonarEfeito(item)
		if item:IsA("ParticleEmitter") or item:IsA("Trail") or item:IsA("Beam") or item:IsA("Light") then
			local effectClone = item:Clone()
			effectClone.Parent = effectPart
			if effectClone:IsA("ParticleEmitter") then
				effectClone.Enabled = true
			end
			table.insert(activeEffects, effectClone)
		end
	end

	if #targetEffect:GetChildren() > 0 or targetEffect:IsA("BasePart") or targetEffect:IsA("Attachment") then
		for _, item in ipairs(targetEffect:GetDescendants()) do
			clonarEfeito(item)
		end
	end
	clonarEfeito(targetEffect)
	AssetBarrageObj:Destroy()

	efeitoAtivoBarrage2 = effectPart

	return {
		Part = effectPart,
		Effects = activeEffects
	}
end

local function pararEfeitoBarrage2(dadosEfeito)
	if not dadosEfeito then return end

	for _, effect in ipairs(dadosEfeito.Effects) do
		if effect:IsA("ParticleEmitter") then
			effect.Enabled = false
		elseif effect:IsA("Trail") or effect:IsA("Beam") or effect:IsA("Light") then
			effect.Enabled = false
		end
	end

	local partParaDeletar = dadosEfeito.Part
	task.delay(3, function()
		if partParaDeletar then
			partParaDeletar:Destroy()
		end
	end)

	if efeitoAtivoBarrage2 == dadosEfeito.Part then
		efeitoAtivoBarrage2 = nil
	end
end

-- [Módulo 10: Barrage 1 Proximity Logic]
local function obterDonoDoSom(som)
	local atual = som.Parent
	while atual and atual ~= workspace do
		if atual:IsA("Model") and atual:FindFirstChildOfClass("Humanoid") then
			return atual
		end
		atual = atual.Parent
	end
	return nil
end

local function estaPertoDeMim(outroChar)
	local meuChar = player.Character
	local meuHrp = meuChar and meuChar:FindFirstChild("HumanoidRootPart")
	if not meuHrp then return false, nil end

	local troncoAlvo = outroChar:FindFirstChild("HumanoidRootPart") 
		or outroChar:FindFirstChild("Torso") 
		or outroChar:FindFirstChild("UpperTorso")
	if not troncoAlvo then return false, nil end

	local distancia = (meuHrp.Position - troncoAlvo.Position).Magnitude
	return distancia <= BARRAGE1_MAX_DISTANCE, troncoAlvo
end

local function criarEfeitoNoAlvo(troncoAlvo)
	local sucesso, AssetBarrage1Obj = pcall(function() return game:GetObjects(HIT_ASSET_ID)[1] end)
	if not sucesso or not AssetBarrage1Obj then return end

	local targetEffect = AssetBarrage1Obj:FindFirstChild(BARRAGE1_EFFECT_NAME, true)
	if not targetEffect then
		AssetBarrage1Obj:Destroy()
		return
	end

	local effectPart = Instance.new("Part")
	effectPart.Size = Vector3.new(1, 1, 1)
	effectPart.Transparency = 1
	effectPart.CanCollide = false
	effectPart.CanTouch = false
	effectPart.CanQuery = false
	effectPart.Massless = true
	effectPart.CFrame = troncoAlvo.CFrame
	effectPart.Parent = workspace

	local weld = Instance.new("WeldConstraint")
	weld.Part0 = effectPart
	weld.Part1 = troncoAlvo
	weld.Parent = effectPart

	effectPart.Anchored = false

	local activeEffects = {}

	local function clonarEfeito(item)
		if item:IsA("ParticleEmitter") or item:IsA("Trail") or item:IsA("Beam") or item:IsA("Light") then
			local effectClone = item:Clone()
			effectClone.Parent = effectPart
			if effectClone:IsA("ParticleEmitter") then
				effectClone.Enabled = true
			end
			table.insert(activeEffects, effectClone)
		end
	end

	if #targetEffect:GetChildren() > 0 or targetEffect:IsA("BasePart") or targetEffect:IsA("Attachment") then
		for _, item in ipairs(targetEffect:GetDescendants()) do
			clonarEfeito(item)
		end
	end
	clonarEfeito(targetEffect)
	AssetBarrage1Obj:Destroy()

	task.wait(0.3)

	for _, effect in ipairs(activeEffects) do
		if effect:IsA("ParticleEmitter") then
			effect.Enabled = false
		elseif effect:IsA("Trail") or effect:IsA("Beam") or effect:IsA("Light") then
			effect.Enabled = false
		end
	end

	task.delay(3, function()
		if effectPart then
			effectPart:Destroy()
		end
	end)
end

local function monitorarSom(som)
	if not som:IsA("Sound") then return end
	if conexoesAtivasSons[som] then return end

	local function verificarEExecutar()
		local somIdString = som.SoundId:match("%d+")
		if somIdString and BARRAGE1_TARGET_SOUNDS[somIdString] then
			local charDono = obterDonoDoSom(som)
			if charDono and charDono ~= player.Character then
				local perto, troncoAlvo = estaPertoDeMim(charDono)
				if perto and troncoAlvo then
					task.spawn(function()
						criarEfeitoNoAlvo(troncoAlvo)
					end)
				end
			end
		end
	end

	local conPlayed = som.Played:Connect(verificarEExecutar)
	local conProp = som:GetPropertyChangedSignal("IsPlaying"):Connect(function()
		if som.IsPlaying then
			verificarEExecutar()
		end
	end)

	conexoesAtivasSons[som] = {conPlayed, conProp}

	som.Destroying:Connect(function()
		if conexoesAtivasSons[som] then
			conexoesAtivasSons[som][1]:Disconnect()
			conexoesAtivasSons[som][2]:Disconnect()
			conexoesAtivasSons[som] = nil
		end
	end)
end

-- [Módulo 11: Playful Cloud Armamento Logic]
local function getConfigCFrame(config, side)
	local offset = config[side .. "Offset"]
	local rot = config[side .. "Rotation"]
	return CFrame.new(offset) * CFrame.Angles(math.rad(rot.X), math.rad(rot.Y), math.rad(rot.Z))
end

local function AttachPlayful(Character)
	local LeftHand = Character:FindFirstChild("LeftHand") or Character:FindFirstChild("Left Arm")
	local RightHand = Character:FindFirstChild("RightHand") or Character:FindFirstChild("Right Arm")
	local Humanoid = Character:FindFirstChildOfClass("Humanoid")
	local Animator = Humanoid and Humanoid:FindFirstChildOfClass("Animator")

	if not LeftHand or not RightHand then return end

	for _,v in ipairs(PlayfulIdleModel:GetDescendants()) do
		if v:IsA("BasePart") then
			v.Anchored = (v == PlayfulLeft or v == PlayfulRight or v == PlayfulMiddle)
			v.CanCollide = false; v.CanTouch = false; v.CanQuery = false; v.Massless = true
		end
	end

	local smoothLeftBase = LeftHand.CFrame
	local smoothRightBase = RightHand.CFrame

	local currentLeftLocal = getConfigCFrame(IDLE_CONFIG, "Left")
	local currentRightLocal = getConfigCFrame(IDLE_CONFIG, "Right")
	local currentMiddleLocal = getConfigCFrame(IDLE_CONFIG, "Middle")

	if PlayfulUpdateConnection then PlayfulUpdateConnection:Disconnect() end

	PlayfulUpdateConnection = RunService.RenderStepped:Connect(function()
		local attackingNow = false
		local blockingNow = false
		local stabilizingNow = false

		if Animator then
			for _, track in ipairs(Animator:GetPlayingAnimationTracks()) do
				local animId = track.Animation and track.Animation.AnimationId:match("%d+$")
				if animId then
					if animId == ESTABILIZAR_ANIM_ID then
						stabilizingNow = true
					elseif animId == BLOCK_ANIM_ID then
						blockingNow = true
					elseif PLAYFUL_ATTACK_ANIMS[animId] then
						attackingNow = true
					end
				end
			end
		end

		local currentTime = os.clock()
		if attackingNow then lastActiveAttackTime = currentTime end
		
		local keepAttackVisual = attackingNow or (currentTime - lastActiveAttackTime < BACK_TO_IDLE_DELAY)
		local keepBlockVisual = blockingNow
		local keepStabilizeVisual = stabilizingNow

		local baseConfig = IDLE_CONFIG
		local useLeftHandAsTarget = false 

		if keepBlockVisual then
			baseConfig = BLOCK_CONFIG
			useLeftHandAsTarget = true 
		elseif keepAttackVisual then
			baseConfig = ATTACK_CONFIG
			useLeftHandAsTarget = true 
		else
			baseConfig = IDLE_CONFIG
			useLeftHandAsTarget = false 
		end

		local targetLeftBase = useLeftHandAsTarget and LeftHand.CFrame or RightHand.CFrame
		local targetRightBase = RightHand.CFrame

		smoothLeftBase = smoothLeftBase:Lerp(targetLeftBase, FOLLOW_SPEED)
		smoothRightBase = smoothRightBase:Lerp(targetRightBase, FOLLOW_SPEED)

		local targetLeftLocal = getConfigCFrame(baseConfig, "Left")
		local targetRightLocal
		if keepStabilizeVisual then
			targetRightLocal = getConfigCFrame(ESTABILIZAR_CONFIG, "Right")
		else
			targetRightLocal = getConfigCFrame(baseConfig, "Right")
		end

		local targetMiddleLocal = getConfigCFrame(baseConfig, "Middle")

		currentLeftLocal = currentLeftLocal:Lerp(targetLeftLocal, LERP_SPEED)
		currentRightLocal = currentRightLocal:Lerp(targetRightLocal, LERP_SPEED)
		currentMiddleLocal = currentMiddleLocal:Lerp(targetMiddleLocal, LERP_SPEED)

		PlayfulLeft.CFrame = smoothLeftBase * currentLeftLocal
		PlayfulRight.CFrame = smoothRightBase * currentRightLocal

		if keepBlockVisual or keepAttackVisual then
			local p0, p1 = PlayfulLeft.Position, PlayfulRight.Position
			local center = (p0 + p1) / 2
			local dist = (p1 - p0).Magnitude
			
			local targetSize = Vector3.new(originalMiddleSize.X, dist, originalMiddleSize.Z)
			PlayfulMiddle.Size = PlayfulMiddle.Size:Lerp(targetSize, LERP_SPEED)
			
			local targetMiddleCFrame = CFrame.lookAt(center, p1) * currentMiddleLocal
			PlayfulMiddle.CFrame = PlayfulMiddle.CFrame:Lerp(targetMiddleCFrame, FOLLOW_SPEED)
		else
			PlayfulMiddle.Size = PlayfulMiddle.Size:Lerp(originalMiddleSize, LERP_SPEED)
			local targetMiddleCFrame = smoothRightBase * currentMiddleLocal
			PlayfulMiddle.CFrame = PlayfulMiddle.CFrame:Lerp(targetMiddleCFrame, FOLLOW_SPEED)
		end
	end)
end

-- [Módulo 2: Domain Invade Cutscene Principal]
local function dispararCutscene()
    if cooldownAtivo then 
        print("[AVISO] Habilidade em Cooldown!")
        return 
    end
    
    cooldownAtivo = true
    tempoRestante = COOLDOWN_TIME
    
    task.spawn(function()
        while tempoRestante > 0 do
            task.wait(0.1)
            tempoRestante = math.max(0, tempoRestante - 0.1)
        end
        cooldownAtivo = false
        print("[Habilidade] Domain Invade está pronta novamente!")
    end)

    local hrp = character:WaitForChild("HumanoidRootPart", 5)
    if not hrp then return end
    
    local cameraReal = Workspace.CurrentCamera
    local playerGui = player:WaitForChild("PlayerGui")

    print("[PROCESSO CRÍTICO] Iniciando Transição Suave...")

    if Workspace:FindFirstChild("Domain_Cinema_Instanciado") then
        Workspace.Domain_Cinema_Instanciado:Destroy()
    end
    if playerGui:FindFirstChild("Cutscene_FadeOverlay") then
        playerGui.Cutscene_FadeOverlay:Destroy()
    end

    local fadeGui = Instance.new("ScreenGui")
    fadeGui.Name = "Cutscene_FadeOverlay"
    fadeGui.IgnoreGuiInset = true
    fadeGui.ResetOnSpawn = false
    fadeGui.Parent = playerGui

    local fadeFrame = Instance.new("Frame")
    fadeFrame.Size = UDim2.new(1, 0, 1, 0)
    fadeFrame.BackgroundColor3 = Color3.new(0, 0, 0)
    fadeFrame.BackgroundTransparency = 1
    fadeFrame.BorderSizePixel = 0
    fadeFrame.Parent = fadeGui

    local entrarPreto = TweenService:Create(fadeFrame, TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundTransparency = 0})
    entrarPreto:Play()
    entrarPreto.Completed:Wait()

    local container = Instance.new("Folder")
    container.Name = "Domain_Cinema_Instanciado"
    container.Parent = Workspace

    local originalAmbient = Lighting.Ambient
    local originalOutdoorAmbient = Lighting.OutdoorAmbient
    local originalClockTime = Lighting.ClockTime
    local originalExposure = Lighting.ExposureCompensation
    local originalFOV = cameraReal.FieldOfView

    Lighting.Ambient = Color3.new(1, 1, 1)
    Lighting.OutdoorAmbient = Color3.new(1, 1, 1)
    Lighting.ClockTime = 14.5
    Lighting.ExposureCompensation = 0
    cameraReal.FieldOfView = 70 

    local skyCustom = Instance.new("Sky")
    local assetFase2 = "rbxassetid://94834946279545"
    skyCustom.SkyboxBk = assetFase2
    skyCustom.SkyboxDn = assetFase2
    skyCustom.SkyboxFt = assetFase2
    skyCustom.SkyboxLf = assetFase2
    skyCustom.SkyboxRt = assetFase2
    skyCustom.SkyboxUp = assetFase2
    skyCustom.Parent = Lighting

    local dofCustom = Instance.new("DepthOfFieldEffect")
    dofCustom.FarIntensity = 1
    dofCustom.FocusDistance = 1
    dofCustom.InFocusRadius = 5
    dofCustom.Parent = Lighting

    local domainClone = targetFolder.Domain:Clone()
    domainClone.Parent = container

    local localizacaoBase = hrp.CFrame
    local cframeAlvoDuble = localizacaoBase

    local rigCinzaOriginal = domainClone:FindFirstChild("Rig")
    if rigCinzaOriginal then
        if rigCinzaOriginal:FindFirstChild("HumanoidRootPart") then
            cframeAlvoDuble = rigCinzaOriginal.HumanoidRootPart.CFrame
        end
        rigCinzaOriginal:Destroy()
    end

    local bgFolder = domainClone:FindFirstChild("BG")
    if bgFolder then
        for _, obj in ipairs(bgFolder:GetChildren()) do
            if obj:IsA("BasePart") or obj:IsA("Model") and obj.Name ~= "Water" then
                local fundoEspelhado = obj:Clone()
                fundoEspelhado.Name = obj.Name .. "_EspelhadoEsquerda"
                fundoEspelhado.Parent = bgFolder
                
                if fundoEspelhado:IsA("BasePart") then
                    local offsetLocal = cframeAlvoDuble:ToObjectSpace(fundoEspelhado.CFrame)
                    local offsetInvertido = CFrame.new(-offsetLocal.X, offsetLocal.Y, offsetLocal.Z) * CFrame.Angles(0, math.pi, 0)
                    fundoEspelhado.CFrame = cframeAlvoDuble:ToWorldSpace(offsetInvertido)
                elseif fundoEspelhado:IsA("Model") then
                    local pivot = fundoEspelhado:GetPivot()
                    local offsetLocal = cframeAlvoDuble:ToObjectSpace(pivot)
                    local offsetInvertido = CFrame.new(-offsetLocal.X, offsetLocal.Y, offsetLocal.Z) * CFrame.Angles(0, math.pi, 0)
                    fundoEspelhado:PivotTo(cframeAlvoDuble:ToWorldSpace(offsetInvertido))
                end
            end
        end
    end

    for _, fx in ipairs(domainClone:GetDescendants()) do
        if fx:IsA("ParticleEmitter") then
            if fx.Name == "Flames" or fx.Name:lower():find("fire") or fx.Name:lower():find("fogo") then
                fx.Enabled = false
            else
                fx.Enabled = true
                fx.Rate = fx.Rate * 1.5
                fx.TimeScale = 0.2
            end
        elseif fx:IsA("Beam") or fx:IsA("Trail") then
            fx.Enabled = true
        elseif fx:IsA("Sound") then
            fx:Play()
        end
    end

    local waterPart = bgFolder and bgFolder:FindFirstChild("Water")
    local regiaoDaAgua = nil

    if waterPart then
        local tamanho = waterPart.Size + Vector3.new(30, 0, 30)
        local cframeAjustado = waterPart.CFrame * CFrame.new(0, -1, 0)
        
        Workspace.Terrain:FillBlock(cframeAjustado, tamanho, Enum.Material.Water)
        regiaoDaAgua = {cframe = cframeAjustado, size = tamanho}
        waterPart.Transparency = 1
    end

    character.Archivable = true
    local npcR6 = character:Clone()
    character.Archivable = false
    
    npcR6.Name = "FollowerNPC_Cinema_Perfeito"
    npcR6.Parent = container

    local npcHumanoid = npcR6:WaitForChild("Humanoid")
    local npcRoot = npcR6:WaitForChild("HumanoidRootPart")

    npcRoot.CFrame = cframeAlvoDuble
    npcR6:ScaleTo(1.8)
    npcRoot.Anchored = true 

    local npcAnimator = npcHumanoid:FindFirstChildOfClass("Animator") or Instance.new("Animator", npcHumanoid)

    for _, v in ipairs(npcR6:GetDescendants()) do
        if v:IsA("LocalScript") or v:IsA("Script") or v:IsA("Tool") or v:IsA("ScreenGui") then
            v:Destroy()
        end
    end

    local cutsceneFinalizada = false
    local loopCamera = nil
    local cameraModel = domainClone:FindFirstChild("Camera")
    local torsoDaCamera = cameraModel and cameraModel:FindFirstChild("Torso")

    if cameraModel and cameraModel:FindFirstChild("Humanoid") then
        local animatorCam = cameraModel.Humanoid:FindFirstChildOfClass("Animator") or Instance.new("Animator", cameraModel.Humanoid)
        local trackCamera = animatorCam:LoadAnimation(targetFolder.Camera)
        trackCamera.Priority = Enum.AnimationPriority.Action4
        trackCamera:Play()
        trackCamera.TimePosition = 2.30 
    end

    if torsoDaCamera then
        cameraReal.CameraType = Enum.CameraType.Scriptable
        cameraReal.CFrame = torsoDaCamera.CFrame
        loopCamera = RunService.RenderStepped:Connect(function()
            if torsoDaCamera and torsoDaCamera.Parent and not cutsceneFinalizada then
                cameraReal.CameraType = Enum.CameraType.Scriptable
                cameraReal.CFrame = torsoDaCamera.CFrame
            else
                if loopCamera then loopCamera:Disconnect() end
            end
        end)
    end

    TweenService:Create(fadeFrame, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {BackgroundTransparency = 1}):Play()

    local function finalizarCutscene()
        if cutsceneFinalizada then return end
        cutsceneFinalizada = true
        
        if loopCamera then loopCamera:Disconnect() end
        
        cameraReal.CameraType = Enum.CameraType.Custom
        cameraReal.CameraSubject = character:FindFirstChildOfClass("Humanoid")
        cameraReal.FieldOfView = originalFOV
        
        if character and character:FindFirstChild("HumanoidRootPart") then
            character:PivotTo(character:GetPivot() * CFrame.new(0, 0, -10))
        end
        
        if skyCustom then skyCustom:Destroy() end
        if dofCustom then dofCustom:Destroy() end
        
        Lighting.Ambient = originalAmbient
        Lighting.OutdoorAmbient = originalOutdoorAmbient
        Lighting.ClockTime = originalClockTime
        Lighting.ExposureCompensation = originalExposure
        
        if regiaoDaAgua then
            Workspace.Terrain:FillBlock(regiaoDaAgua.cframe, regiaoDaAgua.size, Enum.Material.Air)
        end
        
        if fadeGui then fadeGui:Destroy() end
        container:Destroy()
    end

    local somClone = targetFolder.SFX:Clone()
    somClone.Volume = 1.5
    somClone.Parent = npcRoot
    somClone:Play()
    somClone.TimePosition = 2.30 

    if npcAnimator then
        pcall(function()
            local customAnim = Instance.new("Animation")
            customAnim.AnimationId = CUSTOM_ANIM_ID
            
            local npcTrack = npcAnimator:LoadAnimation(customAnim)
            npcTrack.Priority = Enum.AnimationPriority.Action4
            npcTrack:Play()
            npcTrack.TimePosition = 2.30 
            
            task.delay(10.2, function()
                if not cutsceneFinalizada then
                    fadeFrame.BackgroundColor3 = Color3.new(1, 1, 1)
                    TweenService:Create(fadeFrame, TweenInfo.new(2.5, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {BackgroundTransparency = 0}):Play()
                end
            end)
            
            npcTrack.Stopped:Connect(function()
                finalizarCutscene()
                
                local flashGui = Instance.new("ScreenGui")
                flashGui.IgnoreGuiInset = true
                flashGui.Parent = playerGui
                
                local flashFrame = Instance.new("Frame")
                flashFrame.Size = UDim2.new(1, 0, 1, 0)
                flashFrame.BackgroundColor3 = Color3.new(1, 1, 1)
                flashFrame.BorderSizePixel = 0
                flashFrame.Parent = flashGui
                
                local sumirBranco = TweenService:Create(flashFrame, TweenInfo.new(0.9, Enum.EasingStyle.Linear), {BackgroundTransparency = 1})
                sumirBranco:Play()
                sumirBranco.Completed:Connect(function()
                    flashGui:Destroy()
                end)
            end)
        end)
    end

    task.delay(12.7, function()
        finalizarCutscene()
    end)
end

-- [Módulo 2: Domain Invade UI Injector]
local function forceInjectButtonDomain()
    local playerGui = player:WaitForChild("PlayerGui", 10)
    local movesetFolder = nil
    
    for _, v in pairs(playerGui:GetDescendants()) do
        if v:IsA("GuiObject") and v.Name == "Cleaving Whirlwind" then
            movesetFolder = v.Parent
            break
        end
    end

    if movesetFolder then
        if movesetFolder:FindFirstChild("DomainInvadeSkill") then
            movesetFolder.DomainInvadeSkill:Destroy()
        end

        local sample = movesetFolder:FindFirstChild("Cleaving Whirlwind")
        if sample then
            local skillFrame = sample:Clone()
            skillFrame.Name = "DomainInvadeSkill"
            
            local originalSize = skillFrame.Size
            skillFrame.Size = UDim2.new(
                originalSize.X.Scale * 0.8, 
                originalSize.X.Offset * 0.8, 
                originalSize.Y.Scale * 0.8, 
                originalSize.Y.Offset * 0.8
            )

            if skillFrame:FindFirstChild("Item") then skillFrame.Item:Destroy() end
            if skillFrame:FindFirstChild("Cooldown") then skillFrame.Cooldown:Destroy() end
            
            local tipLabel = skillFrame:FindFirstChild("Tip")
            if tipLabel then
                tipLabel.TextSize = 12 
                tipLabel.TextColor3 = Color3.fromRGB(255, 65, 65) 
                tipLabel.Visible = true
            end

            local itemNameBtn = skillFrame:FindFirstChild("ItemName")
            local keyFrame = skillFrame:FindFirstChild("Key")
            local keyLabel = keyFrame and keyFrame:FindFirstChild("Key")

            if itemNameBtn and itemNameBtn:IsA("TextButton") then
                local cloneProps = {
                    Size = itemNameBtn.Size,
                    Position = itemNameBtn.Position,
                    Font = itemNameBtn.Font,
                    TextSize = itemNameBtn.TextSize
                }
                itemNameBtn:Destroy()

                local newBtn = Instance.new("TextButton")
                newBtn.Name = "ItemName"
                newBtn.Size = cloneProps.Size
                newBtn.Position = cloneProps.Position
                newBtn.Font = cloneProps.Font
                newBtn.TextSize = cloneProps.TextSize
                newBtn.BackgroundTransparency = 1 
                newBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
                newBtn.TextWrapped = true
                newBtn.Parent = skillFrame
                
                newBtn.MouseButton1Click:Connect(dispararCutscene)
                itemNameBtn = newBtn
            end

            skillFrame.LayoutOrder = 99
            skillFrame.Parent = movesetFolder

            RunService.RenderStepped:Connect(function()
                if skillFrame and skillFrame.Parent then
                    if itemNameBtn and itemNameBtn.Text ~= "Domain Invade" then
                        itemNameBtn.Text = "Domain Invade"
                    end
                    if keyLabel and keyLabel.Text ~= "C" then
                        keyLabel.Text = "C"
                    end
                    
                    if tipLabel then
                        if cooldownAtivo and tempoRestante > 0 then
                            tipLabel.Text = string.format("%.1fs", tempoRestante)
                            tipLabel.Visible = true
                        else
                            tipLabel.Text = ""
                            tipLabel.Visible = false
                        end
                    end
                end
            end)
        end
    end
end

-- [Módulo 8 & 9: Monitoramento de Animações (Hits, Barrage 2)]
local function monitorarAnimacoesHitsBarrage(charToMonitor)
	local humanoid = charToMonitor:WaitForChild("Humanoid", 10)
	local animator = humanoid and humanoid:WaitForChild("Animator", 10)
	if not animator then return end

	animator.AnimationPlayed:Connect(function(animationTrack)
		local animIdString = animationTrack.Animation.AnimationId:match("%d+")
		
		-- Lógica do Hit (Módulo 8)
		if animIdString and HIT_TARGET_ANIMS[animIdString] then
			local meuHrp = charToMonitor:FindFirstChild("HumanoidRootPart")
			if meuHrp then
				task.spawn(function()
					criarEfeitoHit(meuHrp)
				end)
			end
		end

		-- Lógica da Barrage 2 (Módulo 9)
		local targetCleanId = BARRAGE2_TARGET_ANIM_ID:match("%d+")
		if animIdString and animIdString == targetCleanId then
			local meuHrp = charToMonitor:FindFirstChild("HumanoidRootPart")
			if not meuHrp then return end

			local dadosEfeito = iniciarEfeitoNoPlayer(meuHrp)

			if dadosEfeito then
				local conexaoEnded
				local conexaoStopped

				local function aoTerminar()
					if conexaoEnded then conexaoEnded:Disconnect() end
					if conexaoStopped then conexaoStopped:Disconnect() end
					pararEfeitoBarrage2(dadosEfeito)
				end

				conexaoEnded = animationTrack.Ended:Connect(aoTerminar)
				conexaoStopped = animationTrack.Stopped:Connect(aoTerminar)
			end
		end
	end)
end

-- [Módulo 12: Asset Clean-up ("SetAssets")]
local function monitorAndCleanSetAssets(charToMonitor)
	while charToMonitor and charToMonitor.Parent do
		local setAssets = charToMonitor:FindFirstChild("SetAssets")

		if setAssets then
			setAssets:Destroy()
			print("SetAssets removido.")
		end

		charToMonitor.ChildAdded:Connect(function(child)
			if child.Name == "SetAssets" then
				task.wait() 
				if child and child.Parent then
					child:Destroy()
					print("Novo SetAssets removido.")
				end
			end
		end)

		break
	end
end

--==========================================================================================
-- INICIALIZADORES DINÂMICOS DO PERSONAGEM (CONFIGURADOS NO SPAWN/RESPAWN)
--==========================================================================================

local function setupCharacter(newChar)
	character = newChar
	
	-- 1. Remoção de SetAssets (Módulo 12)
	task.spawn(monitorAndCleanSetAssets, newChar)

	-- 2. Sistema de Som (Módulo 1)
	for _, obj in ipairs(newChar:GetDescendants()) do
		hookSound(obj)
	end
	newChar.DescendantAdded:Connect(hookSound)

	-- 3. Aplicação da Aura (Módulo 3)
	task.spawn(function()
		task.wait(0.5)
		applyAura(newChar)
	end)

	-- 4. Customização Visual de Pele e Roupas (Módulo 4)
	task.spawn(function()
		applySkin(newChar)
	end)

	-- 5. Animação de Idle Infinita (Módulo 7)
	task.spawn(aplicarAnimacao, newChar)

	-- 6. Monitor de Animações Ativas para Efeitos Físicos (Módulo 8 & 9)
	task.spawn(monitorarAnimacoesHitsBarrage, newChar)

	-- 7. Anexação do Modelo Armamento Playful Cloud (Módulo 11)
	task.spawn(function()
		task.wait(1)
		AttachPlayful(newChar)
	end)

	-- 8. Re-Injeção de Botões na UI (Módulos 2 & 5)
	task.spawn(function()
		task.wait(0.4)
		forceInjectButton()
		forceInjectButtonDomain()
	end)
end

-- Inicializa imediatamente no Character atual
if player.Character then
	setupCharacter(player.Character)
end

-- Conecta para aplicar toda a cadeia de scripts automaticamente nos respawns futuros
player.CharacterAdded:Connect(setupCharacter)

-- Detecção do carregamento de aparência nativa para evitar que o Roblox sobrescreva o Skin Changer
player.CharacterAppearanceLoaded:Connect(function(loadedChar)
	applySkin(loadedChar)
end)

--==========================================================================================
-- MONITORAMENTOS TIPO LOOPS GLOBAIS & SEVIÇOS (RUNNING IN BACKGROUND)
--==========================================================================================

-- [Módulo 6: Loop de Atualização TP Walk & Ghost Trail]
task.spawn(function()
	while true do
		local curChar = player.Character
		local humanoid = curChar and curChar:FindFirstChildOfClass("Humanoid")
		local hrp = curChar and curChar:FindFirstChild("HumanoidRootPart")

		if humanoid and hrp and humanoid.MoveDirection.Magnitude > 0 then
			local timeNow = os.clock()

			if timeNow - lastTP >= TP_INTERVAL then
				lastTP = timeNow
				local moveDir = humanoid.MoveDirection

				if moveDir.Magnitude > 0 then
					local startTime = os.clock()
					while os.clock() - startTime < BOOST_TIME do
						hrp.CFrame = hrp.CFrame + (moveDir.Unit * TP_SPEED)
						task.wait()
					end
				end
			end

			if timeNow - lastGhost >= GHOST_INTERVAL then
				lastGhost = timeNow
				CreateGhost()
			end
		else
			lastTP = os.clock()
			lastGhost = os.clock()
		end
		task.wait()
	end
end)

-- [Módulo 10: Inicialização de Detecção de Sons Próximos (Barrage 1)]
task.spawn(function()
	for _, desc in ipairs(workspace:GetDescendants()) do
		monitorarSom(desc)
	end
	workspace.DescendantAdded:Connect(monitorarSom)
end)

--==========================================================================================
-- MAPEAMENTO DE ENTRADA DO TECLADO (TECLAS FÍSICAS DE TRASPORTE E SUPORTE)
--==========================================================================================

UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    
    -- Tecla C para Domínio (Módulo 2)
    if input.KeyCode == Enum.KeyCode.C then
        dispararCutscene()
    end

    -- Tecla X para Flash Step (Módulo 5)
    if input.KeyCode == Enum.KeyCode.X then
        dispararTeleporteFrente()
    end
end)
--==========================================================================================
-- SUPRESSÃO E RETORNO AUTOMÁTICO DO PLAYFUL CLOUD POR ANIMAÇÃO E RESPAWN
--==========================================================================================
task.spawn(function()
    local targetAnimSuppres = "108695775669287"
    
    local function monitorSuppression(char)
        -- Sempre que você nascer, garante que a arma volte para o Workspace para o Módulo 11 funcionar
        if PlayfulAsset and PlayfulAsset.Parent ~= workspace then
            PlayfulAsset.Parent = workspace
            print("[Playful Cloud] Personagem resetado. Arma restaurada para as mãos.")
        end

        local humanoid = char:WaitForChild("Humanoid", 10)
        local animator = humanoid and humanoid:WaitForChild("Animator", 10)
        if not animator then return end
        
        local suppressionConnection
        suppressionConnection = animator.AnimationPlayed:Connect(function(track)
            local animIdStr = track.Animation.AnimationId:match("%d+$")
            if animIdStr and animIdStr == targetAnimSuppres then
                print("[Playful Cloud] Animação detectada. Ocultando arma até a próxima vida...")
                
                -- Desliga o loop do Módulo 11 que fica colando a arma na sua mão
                if PlayfulUpdateConnection then
                    PlayfulUpdateConnection:Disconnect()
                    PlayfulUpdateConnection = nil
                end
                
                -- Remove do Workspace (esconde)
                if PlayfulAsset then
                    PlayfulAsset.Parent = ReplicatedStorage
                end
                
                -- Desconecta esse monitoramento específico temporariamente (para não bugar)
                if suppressionConnection then
                    suppressionConnection:Disconnect()
                end
            end
        end)
    end

    if player.Character then monitorSuppression(player.Character) end
    player.CharacterAdded:Connect(monitorSuppression)
end)
print("Script unificado carregado com total sucesso!")
-- Execução final do arquivo externo contido no fim do script base
local code = readfile("Shibuya.txt")
loadstring(code)()
