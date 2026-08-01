local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

-- Remove GUI anterior se existir
if PlayerGui:FindFirstChild("DesenvolvimentoGUI") then
    PlayerGui.DesenvolvimentoGUI:Destroy()
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "DesenvolvimentoGUI"
ScreenGui.IgnoreGuiInset = true
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = PlayerGui

local Frame = Instance.new("Frame")
Frame.Size = UDim2.new(1, 0, 1, 0)
Frame.Position = UDim2.new(0, 0, 0, 0)
Frame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
Frame.BorderSizePixel = 0
Frame.BackgroundTransparency = 0
Frame.Parent = ScreenGui

local TextLabel = Instance.new("TextLabel")
TextLabel.Size = UDim2.new(0.9, 0, 0.4, 0)
TextLabel.Position = UDim2.new(0.05, 0, 0.3, 0)
TextLabel.BackgroundTransparency = 1
TextLabel.Text = "Este script está em desenvolvimento\nSe acalma, o script vai sair\nPera aí moço"
TextLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
TextLabel.TextTransparency = 0
TextLabel.TextScaled = true
TextLabel.Font = Enum.Font.GothamBold
TextLabel.TextStrokeTransparency = 0.5
TextLabel.Parent = Frame

-- Espera 3.5 segundos e faz o fade out
task.wait(3.5)

local fadeInfo = TweenInfo.new(1.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)

local fadeFrame = TweenService:Create(Frame, fadeInfo, {BackgroundTransparency = 1})
local fadeText = TweenService:Create(TextLabel, fadeInfo, {TextTransparency = 1, TextStrokeTransparency = 1})

fadeFrame:Play()
fadeText:Play()

fadeFrame.Completed:Connect(function()
    ScreenGui:Destroy()
end)
