```lua
--local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

local M1Event = ReplicatedStorage:WaitForChild("RemoteEvents"):WaitForChild("M1Event")

local DAMAGE_PER_HIT = {8, 8, 10, 16}
local HITBOX_RANGE = 6

-- Animações do combo (ordem correta: soco1, soco2, chute, finalizador)
-- Animações originais criadas pelo usuário (acesso compartilhado confirmado).
local COMBO_ANIMS = {
	[1] = 106073313723061, -- soco 1
	[2] = 120139818836669, -- soco 2
	[3] = 104052955770282, -- chute
	[4] = 131113761034953, -- finalizador
}

local function playComboAnim(character, comboId)
	print(">>> playComboAnim chamado, comboId:", comboId)

	local humanoid = character:FindFirstChildOfClass("Humanoid")
	if not humanoid then
		print(">>> ERRO: Humanoid não encontrado")
		return
	end

	local animId = COMBO_ANIMS[comboId]
	print(">>> animId encontrado:", animId)

	if animId and animId ~= 0 then
		local anim = Instance.new("Animation")
		anim.AnimationId = "rbxassetid://" .. animId

		local animator = humanoid:WaitForChild("Animator")
		local track = animator:LoadAnimation(anim)

		print(">>> Track criado, tentando dar Play")
		track.Priority = Enum.AnimationPriority.Action
		track:Play(0.05)
		print(">>> Play executado")

		anim:Destroy()
	end
end

local function performHit(player, comboCount)
	local character = player.Character
	if not character then return end

	local rootPart = character:FindFirstChild("HumanoidRootPart")
	local humanoid = character:FindFirstChild("Humanoid")
	if not rootPart or not humanoid or humanoid.Health <= 0 then return end

	comboCount = math.clamp(math.floor(tonumber(comboCount) or 1), 1, 4)
	local damage = DAMAGE_PER_HIT[comboCount]

	-- Toca a animação no servidor (replica para todos os clientes)
	playComboAnim(character, comboCount)

	for _, obj in workspace:GetChildren() do
		if obj:IsA("Model") and obj ~= character then
			local otherRoot = obj:FindFirstChild("HumanoidRootPart")
			local otherHumanoid = obj:FindFirstChildOfClass("Humanoid")

			if otherRoot and otherHumanoid and otherHumanoid.Health > 0 then
				local distance = (otherRoot.Position - rootPart.Position).Magnitude
				local direction = (otherRoot.Position - rootPart.Position).Unit
				local facing = rootPart.CFrame.LookVector
				local dot = facing:Dot(direction)

				if distance <= HITBOX_RANGE and dot > 0.5 then
					otherHumanoid:TakeDamage(damage)

					if comboCount == 4 then
						local knockDir = direction * 25
						otherRoot.Velocity = Vector3.new(knockDir.X, 15, knockDir.Z)
					end
				end
			end
		end
	end

	local lunge = Instance.new("BodyVelocity")
	lunge.MaxForce = Vector3.new(4000, 0, 4000)
	lunge.Velocity = rootPart.CFrame.LookVector * 12
	lunge.Parent = rootPart
	game:GetService("Debris"):AddItem(lunge, 0.1)
end

M1Event.OnServerEvent:Connect(function(player, comboId)
	performHit(player, comboId)
end)
```
