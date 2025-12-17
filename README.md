(function()
    -- ═══════════════════════════════════════════════════════════
    -- SISTEMA UNIVERSAL - iOS/PC/Android/Samsung Compatible
    -- Versão 3.1 - Interface Limpa + Notificações Rápidas
    -- ═══════════════════════════════════════════════════════════
    
    local HttpService = game:GetService("HttpService")
    local TeleportService = game:GetService("TeleportService")
    local TweenService = game:GetService("TweenService")
    local Players = game:GetService("Players")
    local RunService = game:GetService("RunService")
    
    -- URLs das APIs
    local url_api1 = table.concat({
        "https://", "yoidkwhatsthis", ".elijahmoses-j", ".workers.dev",
        "/api/", "messages"
    })
    
    local url_api2 = table.concat({
        "https://", "brainrot-finder", "-default-rtdb", ".firebaseio.com",
        "/brainrots/", "latest.json"
    })
    
    -- Configurações
    local cfg_gameid = 109983668079237
    local cfg_lifetime = 15
    local cfg_maxcards = 3
    local cfg_interval_api = 1
    local cfg_interval_local = 0.5
    local cfg_min_generation = 50
    
    -- Variáveis globais
    local ui_gui, ui_frame
    local tbl_cards = {}
    local tbl_seen = {}
    local last_id = nil
    local is_busy = false
    local min_target_value = 0
    local current_mode = "DETECTING"
    local http_available = false
    
    -- ═══════════════════════════════════════════════════════════
    -- DETECÇÃO DE PLATAFORMA E CAPACIDADES
    -- ═══════════════════════════════════════════════════════════
    
    local function detect_platform()
        local platform = "UNKNOWN"
        local user_input = game:GetService("UserInputService")
        
        if user_input.TouchEnabled and not user_input.KeyboardEnabled then
            if user_input.AccelerometerEnabled then
                platform = "MOBILE"
            else
                platform = "TABLET"
            end
        elseif user_input.KeyboardEnabled and user_input.MouseEnabled then
            platform = "PC"
        end
        
        return platform
    end
    
    local function test_http_access()
        local methods = {
            function(url)
                return game:HttpGet(url)
            end,
            function(url)
                return HttpService:GetAsync(url)
            end,
            function(url)
                if syn and syn.request then
                    local r = syn.request({Url=url, Method="GET"})
                    if r.Success and r.StatusCode == 200 then
                        return r.Body
                    end
                end
                return nil
            end,
            function(url)
                if request then
                    local r = request({Url=url, Method="GET"})
                    if r.Success and r.StatusCode == 200 then
                        return r.Body
                    end
                end
                return nil
            end
        }
        
        for idx, method in ipairs(methods) do
            local ok, result = pcall(function()
                return method(url_api2)
            end)
            
            if ok and result and result ~= "null" and result ~= "" then
                print("✓ HTTP disponível - Método", idx)
                return method
            end
        end
        
        return nil
    end
    
    -- ═══════════════════════════════════════════════════════════
    -- FUNÇÕES UTILITÁRIAS
    -- ═══════════════════════════════════════════════════════════
    
    local function format_number(num)
        if num >= 1000000000 then
            return string.format("%.2fB", num / 1000000000)
        elseif num >= 1000000 then
            return string.format("%.2fM", num / 1000000)
        elseif num >= 1000 then
            return string.format("%.1fK", num / 1000)
        else
            return tostring(math.floor(num))
        end
    end
    
    local function parse_money_value(text)
        if not text then return 0 end
        
        text = tostring(text):gsub("%*", ""):gsub("$", ""):gsub(",", ""):gsub(" ", "")
        local number = text:match("([%d%.]+)")
        if not number then return 0 end
        
        number = tonumber(number) or 0
        local suffix = text:match("[kKmMbB]")
        
        if suffix then
            suffix = suffix:lower()
            if suffix == "k" then return number * 1000
            elseif suffix == "m" then return number * 1000000
            elseif suffix == "b" then return number * 1000000000
            end
        end
        
        return number
    end
    
    local function parse_generation(text)
        if not text then return 0 end
        local num = tonumber(text:match("%d+"))
        return num or 0
    end
    
    -- ═══════════════════════════════════════════════════════════
    -- MODO 1: APIS EXTERNAS (PC/Android)
    -- ═══════════════════════════════════════════════════════════
    
    local http_method = nil
    
    local function fetch_api_data(url, method)
        if not method then return nil end
        
        local ok, result = pcall(function()
            return method(url)
        end)
        
        if not ok or not result or result == "" or result == "null" then
            return nil
        end
        
        local decode_ok, data = pcall(function()
            return HttpService:JSONDecode(result)
        end)
        
        if not decode_ok then return nil end
        return data
    end
    
    local function parse_discord_embed(embed)
        local info = {
            name = "Unknown",
            money = "Unknown",
            job = "Unknown",
            generation = 0,
            moneyVal = 0,
            source = "API"
        }
        
        if not embed or not embed.fields then return info end
        
        for _, field in pairs(embed.fields) do
            local field_name = field.name:lower()
            local field_value = field.value or ""
            
            if field_name:find("name") or field_name:find("🏷️") then
                info.name = field_value:gsub("%*", ""):match("^%s*(.-)%s*$")
            elseif field_name:find("money") or field_name:find("💰") then
                info.money = field_value:gsub("%*", ""):match("^%s*(.-)%s*$")
                info.moneyVal = parse_money_value(field_value)
            elseif field_name:find("job") or field_name:find("🆔") then
                info.job = field_value:gsub("```", ""):gsub("`", ""):match("^%s*(.-)%s*$")
            elseif field_name:find("generation") then
                info.generation = parse_generation(field_value)
            end
        end
        
        return info
    end
    
    local function monitor_apis()
        while http_available do
            local data2 = fetch_api_data(url_api2, http_method)
            if data2 and data2.name and data2.generation then
                local brainrot_id = tostring(data2.name) .. tostring(data2.generation)
                
                if brainrot_id ~= last_id then
                    last_id = brainrot_id
                    
                    local gen_text = tostring(data2.generation)
                    local money_val = parse_money_value(gen_text)
                    local job_id = data2.jobId or "Unknown"
                    local gen_num = parse_generation(gen_text)
                    
                    local server_data = {
                        name = tostring(data2.name),
                        money = gen_text,
                        job = job_id,
                        generation = gen_num,
                        moneyVal = money_val,
                        source = "API2"
                    }
                    
                    process_new_server(server_data)
                end
            end
            
            local data1 = fetch_api_data(url_api1, http_method)
            if data1 and type(data1) == "table" then
                for _, message in pairs(data1) do
                    if message.embeds and #message.embeds > 0 then
                        local server_data = parse_discord_embed(message.embeds[1])
                        if server_data.name ~= "Unknown" then
                            process_new_server(server_data)
                            break
                        end
                    end
                end
            end
            
            task.wait(cfg_interval_api)
        end
    end
    
    -- ═══════════════════════════════════════════════════════════
    -- MODO 2: SCANNING LOCAL (iOS/Fallback Universal)
    -- ═══════════════════════════════════════════════════════════
    
    local function find_brainrots_in_workspace()
        local workspace = game:GetService("Workspace")
        local plots = workspace:FindFirstChild("Plots")
        if not plots then return {} end
        
        local found_brainrots = {}
        
        for _, plot in ipairs(plots:GetChildren()) do
            local podiums = plot:FindFirstChild("AnimalPodiums")
            if podiums then
                for _, podium in ipairs(podiums:GetChildren()) do
                    local base = podium:FindFirstChild("Base")
                    if base then
                        local spawn = base:FindFirstChild("Spawn")
                        if spawn then
                            local attach = spawn:FindFirstChild("Attachment")
                            if attach then
                                local overhead = attach:FindFirstChild("AnimalOverhead")
                                if overhead then
                                    local name_lbl = overhead:FindFirstChild("DisplayName")
                                    local gen_lbl = overhead:FindFirstChild("Generation")
                                    local rare_lbl = overhead:FindFirstChild("Rarity")
                                    
                                    if name_lbl and gen_lbl then
                                        local pet_name = name_lbl.Text
                                        local gen_text = gen_lbl.Text
                                        local rarity = rare_lbl and rare_lbl.Text or "Common"
                                        
                                        local gen_num = parse_generation(gen_text)
                                        local money_val = parse_money_value(gen_text)
                                        
                                        if not pet_name:lower():find("fusing") and gen_num > 0 then
                                            table.insert(found_brainrots, {
                                                name = pet_name,
                                                generation = gen_num,
                                                money = gen_text,
                                                moneyVal = money_val,
                                                rarity = rarity,
                                                position = spawn.Position,
                                                part = spawn,
                                                job = "LOCAL",
                                                source = "LOCAL"
                                            })
                                        end
                                    end
                                end
                            end
                        end
                    end
                end
            end
        end
        
        return found_brainrots
    end
    
    local function find_best_local_brainrot()
        local brainrots = find_brainrots_in_workspace()
        if #brainrots == 0 then return nil end
        
        table.sort(brainrots, function(a, b)
            if a.moneyVal > 0 and b.moneyVal > 0 then
                return a.moneyVal > b.moneyVal
            end
            return a.generation > b.generation
        end)
        
        return brainrots[1]
    end
    
    local function monitor_local_workspace()
        local last_alert_id = nil
        
        while not http_available do
            local best = find_best_local_brainrot()
            
            if best and best.generation >= cfg_min_generation then
                local alert_id = best.name .. best.generation
                
                if alert_id ~= last_alert_id then
                    last_alert_id = alert_id
                    
                    if best.moneyVal >= min_target_value or best.generation >= cfg_min_generation then
                        process_new_server(best)
                    end
                end
            end
            
            task.wait(cfg_interval_local)
        end
    end
    
    -- ═══════════════════════════════════════════════════════════
    -- SISTEMA DE TELEPORTE
    -- ═══════════════════════════════════════════════════════════
    
    local function is_valid_jobid(job_id)
        if not job_id or type(job_id) ~= "string" or job_id == "LOCAL" or job_id == "Unknown" then
            return false
        end
        
        job_id = job_id:match("^%s*(.-)%s*$")
        
        if #job_id < 10 then return false end
        if not job_id:match("^[%w%-]+$") then return false end
        
        return true
    end
    
    local function teleport_to_server(job_id, card)
        if not is_valid_jobid(job_id) then
            if card then
                TweenService:Create(card, TweenInfo.new(0.3), {BackgroundColor3 = Color3.fromRGB(150, 30, 30)}):Play()
                task.wait(0.6)
                TweenService:Create(card, TweenInfo.new(0.3), {BackgroundColor3 = Color3.fromRGB(20, 20, 20)}):Play()
            end
            return false
        end
        
        if card then
            TweenService:Create(card, TweenInfo.new(0.3), {BackgroundColor3 = Color3.fromRGB(255, 215, 0)}):Play()
        end
        
        local ok, err = pcall(function()
            TeleportService:TeleportToPlaceInstance(cfg_gameid, job_id, Players.LocalPlayer)
        end)
        
        if not ok then
            if card then
                task.wait(0.5)
                TweenService:Create(card, TweenInfo.new(0.3), {BackgroundColor3 = Color3.fromRGB(150, 30, 30)}):Play()
            end
            return false
        end
        
        return true
    end
    
    local function teleport_to_local_pet(pet_data, card)
        if not pet_data or not pet_data.part then
            if card then
                TweenService:Create(card, TweenInfo.new(0.3), {BackgroundColor3 = Color3.fromRGB(150, 30, 30)}):Play()
            end
            return false
        end
        
        if card then
            TweenService:Create(card, TweenInfo.new(0.3), {BackgroundColor3 = Color3.fromRGB(0, 255, 0)}):Play()
        end
        
        local player = Players.LocalPlayer
        local char = player.Character
        if not char then return false end
        
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not hrp then return false end
        
        local target_pos = pet_data.position + Vector3.new(0, 3, 5)
        hrp.CFrame = CFrame.new(target_pos)
        
        return true
    end
    
    -- ═══════════════════════════════════════════════════════════
    -- SISTEMA DE CARDS (UI)
    -- ═══════════════════════════════════════════════════════════
    
    local function remove_card(card)
        if not card or not card.Parent then return end
        
        for _, child in pairs(card:GetChildren()) do
            if child:IsA("TextLabel") then
                TweenService:Create(child, TweenInfo.new(0.2), {TextTransparency = 1}):Play()
            elseif child:IsA("UIStroke") then
                TweenService:Create(child, TweenInfo.new(0.2), {Transparency = 1}):Play()
            end
        end
        
        TweenService:Create(card, TweenInfo.new(0.2), {BackgroundTransparency = 1}):Play()
        
        task.wait(0.2)
        
        for idx, card_data in ipairs(tbl_cards) do
            if card_data.card == card then
                table.remove(tbl_cards, idx)
                break
            end
        end
        
        card:Destroy()
        
        task.wait(0.05)
        for idx, card_data in ipairs(tbl_cards) do
            local target_pos = UDim2.new(0, 10, 0, (idx-1) * 55 + 10)
            TweenService:Create(
                card_data.card, 
                TweenInfo.new(0.25, Enum.EasingStyle.Quad), 
                {Position = target_pos}
            ):Play()
        end
    end
    
    local function remove_oldest_card()
        if #tbl_cards > 0 then
            remove_card(tbl_cards[1].card)
            task.wait(0.25)
        end
    end
    
    local function create_card(server_data)
        while is_busy do
            task.wait(0.1)
        end
        
        is_busy = true
        
        if #tbl_cards >= cfg_maxcards then
            remove_oldest_card()
        end
        
        local card_idx = #tbl_cards + 1
        
        local card = Instance.new("Frame")
        card.Name = "BrainrotCard_" .. server_data.source
        card.Size = UDim2.new(1, -20, 0, 50)
        card.Position = UDim2.new(0, 10, 0, (card_idx-1) * 55 + 10)
        card.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
        card.BorderSizePixel = 0
        card.BackgroundTransparency = 1
        card.Parent = ui_frame
        
        local card_corner = Instance.new("UICorner", card)
        card_corner.CornerRadius = UDim.new(0, 8)
        
        local border_color
        if server_data.source == "API1" or server_data.source == "API2" then
            border_color = Color3.fromRGB(255, 215, 0)
        else
            border_color = Color3.fromRGB(0, 255, 100)
        end
        
        local stroke = Instance.new("UIStroke", card)
        stroke.Color = border_color
        stroke.Thickness = 2
        stroke.Transparency = 1
        
        local source_label = Instance.new("TextLabel", card)
        source_label.Size = UDim2.new(0, 60, 0, 14)
        source_label.Position = UDim2.new(0, 6, 0, 3)
        source_label.BackgroundTransparency = 1
        source_label.Text = server_data.source
        source_label.TextColor3 = border_color
        source_label.Font = Enum.Font.GothamBold
        source_label.TextSize = 10
        source_label.TextXAlignment = Enum.TextXAlignment.Left
        source_label.TextTransparency = 1
        
        local name_label = Instance.new("TextLabel", card)
        name_label.Size = UDim2.new(0.55, -15, 0, 20)
        name_label.Position = UDim2.new(0, 6, 0, 20)
        name_label.BackgroundTransparency = 1
        name_label.Text = server_data.name
        name_label.TextColor3 = Color3.fromRGB(255, 255, 255)
        name_label.Font = Enum.Font.GothamBold
        name_label.TextSize = 13
        name_label.TextXAlignment = Enum.TextXAlignment.Left
        name_label.TextTruncate = Enum.TextTruncate.AtEnd
        name_label.TextTransparency = 1
        
        local display_money = server_data.moneyVal > 0 
            and format_number(server_data.moneyVal) 
            or server_data.money
        
        local money_label = Instance.new("TextLabel", card)
        money_label.Size = UDim2.new(0.45, -15, 0, 20)
        money_label.Position = UDim2.new(0.55, 0, 0, 20)
        money_label.BackgroundTransparency = 1
        money_label.Text = display_money
        money_label.TextColor3 = border_color
        money_label.Font = Enum.Font.GothamBold
        money_label.TextSize = 14
        money_label.TextXAlignment = Enum.TextXAlignment.Right
        money_label.TextTransparency = 1
        
        local click_button = Instance.new("TextButton", card)
        click_button.Size = UDim2.new(1, 0, 1, 0)
        click_button.BackgroundTransparency = 1
        click_button.Text = ""
        click_button.ZIndex = 2
        
        local card_data = {
            card = card,
            data = server_data,
            createdAt = tick()
        }
        table.insert(tbl_cards, card_data)
        
        TweenService:Create(card, TweenInfo.new(0.3, Enum.EasingStyle.Back), {BackgroundTransparency = 0}):Play()
        TweenService:Create(stroke, TweenInfo.new(0.3), {Transparency = 0}):Play()
        TweenService:Create(source_label, TweenInfo.new(0.3), {TextTransparency = 0}):Play()
        TweenService:Create(name_label, TweenInfo.new(0.3), {TextTransparency = 0}):Play()
        TweenService:Create(money_label, TweenInfo.new(0.3), {TextTransparency = 0}):Play()
        
        click_button.MouseButton1Click:Connect(function()
            if server_data.source == "LOCAL" then
                teleport_to_local_pet(server_data, card)
            else
                teleport_to_server(server_data.job, card)
            end
        end)
        
        click_button.MouseEnter:Connect(function()
            TweenService:Create(card, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(35, 35, 35)}):Play()
        end)
        
        click_button.MouseLeave:Connect(function()
            TweenService:Create(card, TweenInfo.new(0.2), {BackgroundColor3 = Color3.fromRGB(20, 20, 20)}):Play()
        end)
        
        task.spawn(function()
            task.wait(cfg_lifetime)
            remove_card(card)
        end)
        
        is_busy = false
    end
    
    function process_new_server(server_data)
        if not server_data then return end
        
        if server_data.moneyVal < min_target_value and server_data.generation < cfg_min_generation then
            return
        end
        
        local unique_id = server_data.name .. (server_data.generation or 0) .. (server_data.job or "")
        if tbl_seen[unique_id] then return end
        tbl_seen[unique_id] = true
        
        create_card(server_data)
        
        print(string.format("📢 [%s] %s | Gen: %s | %s", 
            server_data.source, 
            server_data.name, 
            server_data.generation or server_data.money,
            server_data.job
        ))
    end
    
    -- ═══════════════════════════════════════════════════════════
    -- INTERFACE GRÁFICA
    -- ═══════════════════════════════════════════════════════════
    
    local function create_ui()
        local player = Players.LocalPlayer
        if not player then return end
        
        local old_ui = player.PlayerGui:FindFirstChild("UniversalBrainrots")
        if old_ui then old_ui:Destroy() end
        
        ui_gui = Instance.new("ScreenGui")
        ui_gui.Name = "UniversalBrainrots"
        ui_gui.ResetOnSpawn = false
        ui_gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
        ui_gui.Parent = player.PlayerGui
        
        local filter_frame = Instance.new("Frame")
        filter_frame.Size = UDim2.new(0, 380, 0, 45)
        filter_frame.Position = UDim2.new(0.5, -190, 0, 10)
        filter_frame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
        filter_frame.BorderSizePixel = 0
        filter_frame.Parent = ui_gui
        
        local filter_corner = Instance.new("UICorner")
        filter_corner.CornerRadius = UDim.new(0, 8)
        filter_corner.Parent = filter_frame
        
        local filter_stroke = Instance.new("UIStroke")
        filter_stroke.Color = Color3.fromRGB(70, 70, 70)
        filter_stroke.Thickness = 1.5
        filter_stroke.Parent = filter_frame
        
        local filter_label = Instance.new("TextLabel")
        filter_label.Size = UDim2.new(0, 60, 0, 30)
        filter_label.Position = UDim2.new(0, 10, 0, 7)
        filter_label.BackgroundTransparency = 1
        filter_label.Text = "Target:"
        filter_label.TextColor3 = Color3.fromRGB(255, 255, 255)
        filter_label.Font = Enum.Font.GothamBold
        filter_label.TextSize = 13
        filter_label.TextXAlignment = Enum.TextXAlignment.Left
        filter_label.Parent = filter_frame
        
        local filter_input = Instance.new("TextBox")
        filter_input.Size = UDim2.new(0, 200, 0, 30)
        filter_input.Position = UDim2.new(0, 75, 0, 7)
        filter_input.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
        filter_input.TextColor3 = Color3.fromRGB(255, 255, 255)
        filter_input.PlaceholderText = "Ex: 10M, 500K, 1B ou Gen50"
        filter_input.PlaceholderColor3 = Color3.fromRGB(150, 150, 150)
        filter_input.Font = Enum.Font.Gotham
        filter_input.TextSize = 12
        filter_input.Text = ""
        filter_input.ClearTextOnFocus = false
        filter_input.Parent = filter_frame
        
        local input_corner = Instance.new("UICorner")
        input_corner.CornerRadius = UDim.new(0, 6)
        input_corner.Parent = filter_input
        
        local input_stroke = Instance.new("UIStroke")
        input_stroke.Color = Color3.fromRGB(90, 90, 90)
        input_stroke.Thickness = 1
        input_stroke.Parent = filter_input
        
        local apply_btn = Instance.new("TextButton")
        apply_btn.Size = UDim2.new(0, 80, 0, 30)
        apply_btn.Position = UDim2.new(0, 285, 0, 7)
        apply_btn.BackgroundColor3 = Color3.fromRGB(0, 150, 255)
        apply_btn.TextColor3 = Color3.fromRGB(255, 255, 255)
        apply_btn.Font = Enum.Font.GothamBold
        apply_btn.TextSize = 12
        apply_btn.Text = "Apply"
        apply_btn.Parent = filter_frame
        
        local btn_corner = Instance.new("UICorner")
        btn_corner.CornerRadius = UDim.new(0, 6)
        btn_corner.Parent = apply_btn
        
        local function parse_user_input(txt)
            if not txt or txt == "" then return 0 end
            
            txt = txt:upper():gsub("%s", "")
            
            if txt:find("GEN") or txt:find("G") then
                local num = txt:match("(%d+)")
                if num then
                    cfg_min_generation = tonumber(num) or 50
                    return 0
                end
            end
            
            local num = txt:match("([%d%.]+)")
            if not num then return 0 end
            
            num = tonumber(num) or 0
            
            if txt:find("K") then
                return num * 1000
            elseif txt:find("M") then
                return num * 1000000
            elseif txt:find("B") then
                return num * 1000000000
            end
            
            return num
        end
        
        apply_btn.MouseButton1Click:Connect(function()
            local input_val = parse_user_input(filter_input.Text)
            min_target_value = input_val
            
            apply_btn.BackgroundColor3 = Color3.fromRGB(0, 255, 100)
            apply_btn.Text = "✓ Applied"
            
            if input_val > 0 then
                print(string.format("🎯 Filtro ativado: Mínimo %s", format_number(input_val)))
            elseif cfg_min_generation > 0 then
                print(string.format("🎯 Filtro ativado: Gen mínima %d", cfg_min_generation))
            else
                print("🎯 Filtro desativado")
            end
            
            task.wait(1)
            apply_btn.BackgroundColor3 = Color3.fromRGB(0, 150, 255)
            apply_btn.Text = "Apply"
        end)
        
        apply_btn.MouseEnter:Connect(function()
            apply_btn.BackgroundColor3 = Color3.fromRGB(0, 180, 255)
        end)
        
        apply_btn.MouseLeave:Connect(function()
            if apply_btn.Text == "Apply" then
                apply_btn.BackgroundColor3 = Color3.fromRGB(0, 150, 255)
            end
        end)
        
        ui_frame = Instance.new("Frame")
        ui_frame.Size = UDim2.new(0, 380, 0, 280)
        ui_frame.Position = UDim2.new(0.5, -190, 0, 65)
        ui_frame.BackgroundTransparency = 1
        ui_frame.BorderSizePixel = 0
        ui_frame.Parent = ui_gui
    end
    
    -- ═══════════════════════════════════════════════════════════
    -- INICIALIZAÇÃO E DETECÇÃO AUTOMÁTICA
    -- ═══════════════════════════════════════════════════════════
    
    local function initialize()
        local player = Players.LocalPlayer
        if not player then
            repeat task.wait(0.5) until Players.LocalPlayer
            player = Players.LocalPlayer
        end
        
        print("═══════════════════════════════════════════════════")
        print("🌍 UNIVERSAL BRAINROT FINDER v3.1")
        print("   iOS | PC | Android | Samsung Compatible")
        print("═══════════════════════════════════════════════════")
        
        local platform = detect_platform()
        print("📱 Plataforma detectada:", platform)
        
        print("🔍 Testando acesso HTTP...")
        http_method = test_http_access()
        
        if http_method then
            http_available = true
            current_mode = "API"
            print("✅ HTTP DISPONÍVEL - Modo API ativado")
            print("   📡 Fontes: Discord Webhook + Firebase")
        else
            http_available = false
            current_mode = "LOCAL"
            print("⚠️  HTTP INDISPONÍVEL - Modo Local ativado")
            print("   🌐 Fonte: Scanning do Workspace")
        end
        
        print("⏱️  Intervalo API:", cfg_interval_api, "segundos")
        print("⏱️  Intervalo Local:", cfg_interval_local, "segundos")
        print("📦 Cards máximos:", cfg_maxcards)
        print("⏳ Duração cards:", cfg_lifetime, "segundos")
        print("🎮 Game ID:", cfg_gameid)
        print("═══════════════════════════════════════════════════")
        
        create_ui()
        
        if http_available then
            task.spawn(monitor_apis)
            print("✅ Monitor de APIs iniciado (Modo Online)")
        else
            task.spawn(monitor_local_workspace)
            print("✅ Monitor Local iniciado (Modo Offline)")
        end
        
        print("✅ Sistema iniciado com sucesso!")
        print("🎯 Aguardando brainrots...")
        
        if current_mode == "API" then
            print("💛 Cards amarelos = APIs Externas")
        else
            print("💚 Cards verdes = Scanning Local")
        end
    end
    
    initialize()
end)()

-- ═══════════════════════════════════════════════════════════
-- PARTE 2: BOTÕES UI COM FUNCIONALIDADES
-- ═══════════════════════════════════════════════════════════

local plr_service = game:GetService("Players")
local uis_service = game:GetService("UserInputService")
local local_plr = plr_service.LocalPlayer

local btn_gui = Instance.new("ScreenGui")
btn_gui.Name = "ControlButtons"
btn_gui.Parent = local_plr:WaitForChild("PlayerGui")

local farm_active = false
local esp_active = false
local farm_thread = nil
local esp_thread = nil

-- ═══════════════════════════════════════════════════════════
-- SCRIPT 1: FARM AUTOMATIZADO
-- ═══════════════════════════════════════════════════════════

local function start_farm()
    if farm_active then return end
    farm_active = true
    
    farm_thread = task.spawn(function()
        local cframe_list = {
            Vector3.new(-478, 14, 219),
            Vector3.new(-478, 13, 113),
            Vector3.new(-478, 14, 6),
            Vector3.new(-478, 14, -102),
            Vector3.new(-341, 13, -99),
            Vector3.new(-341, 14, 8),
            Vector3.new(-341, 13, 114),
            Vector3.new(-341, 14, 221),
        }
        
        local plrs = game:GetService('Players')
        local rs = game:GetService('ReplicatedStorage')
        local p = plrs.LocalPlayer
        
        local function get_char_parts()
            local chr = p.Character or p.CharacterAdded:Wait()
            local root = chr:WaitForChild('HumanoidRootPart', 2)
            local hum = chr:FindFirstChildOfClass('Humanoid')
            return chr, hum, root
        end
        
        local c, h, hrp = get_char_parts()
        
        local coil_name = 'Coil Combo'
        local grapple_name, carpet_name = 'Grapple Hook', 'Flying Carpet'
        local remote_path = rs:WaitForChild('Packages'):WaitForChild('Net'):WaitForChild('RE/UseItem')
        local args_tbl = {0.28693917592366536}
        local ascend_h = 25
        
        local function equip_tool(tool_name)
            c, h, hrp = get_char_parts()
            local t = p.Backpack:FindFirstChild(tool_name) or c:FindFirstChild(tool_name)
            if t then
                h:EquipTool(t)
                return t
            end
            return nil
        end
        
        local function parse_gen_num(txt)
            txt = txt:gsub('[%$%s]', '')
            local num_part, sfx = txt:match('([%d%.]+)([KMB]?)')
            num_part = tonumber(num_part) or 0
            local mult = (sfx == 'K' and 1e3) or (sfx == 'M' and 1e6) or (sfx == 'B' and 1e9) or 1
            return num_part * mult
        end
        
        local function find_highest_gen()
            local best_val = -math.huge
            local best_info = nil
            for _, obj in ipairs(workspace:GetDescendants()) do
                if obj:IsA('TextLabel') and obj.Name == 'Generation' then
                    if not string.find(obj.Text:lower(), 'fusing') then
                        local val = parse_gen_num(obj.Text)
                        if val > best_val then
                            local disp_obj = obj.Parent:FindFirstChild('DisplayName')
                            local rare_obj = obj.Parent:FindFirstChild('Rarity')
                            if not (disp_obj and string.find(disp_obj.Text:lower(), 'fusing')) then
                                local par, mdl = obj.Parent, nil
                                while par and par ~= workspace do
                                    if par:IsA('Model') and par.Parent and par.Parent.Name == 'Plots' then
                                        mdl = par
                                        break
                                    end
                                    par = par.Parent
                                end
                                
                                local prim_part = mdl and (mdl.PrimaryPart or mdl:FindFirstChildWhichIsA('BasePart'))
                                best_val = val
                                best_info = {
                                    displayName = disp_obj and disp_obj.Text or 'N/A',
                                    generation = obj.Text,
                                    rarity = rare_obj and rare_obj.Text or 'N/A',
                                    value = val,
                                    model = mdl,
                                    primaryPart = prim_part,
                                }
                            end
                        end
                    end
                end
            end
            return best_info
        end
        
        local function find_nearest_cf(pos)
            local closest, dist = nil, math.huge
            for _, v in ipairs(cframe_list) do
                local d = (Vector3.new(v.X, pos.Y, v.Z) - pos).Magnitude
                if d < dist then
                    dist = d
                    closest = v
                end
            end
            return closest
        end
        
        local function tp_sequence(tp_pos)
            c, h, hrp = get_char_parts()
            if not hrp then return end
            
            local coil = equip_tool(coil_name)
            if coil then
                hrp.AssemblyLinearVelocity = Vector3.new(0, 75, 0)
            end
            
            local target_y = hrp.Position.Y + ascend_h
            repeat task.wait() until hrp.Position.Y >= target_y
            
            hrp.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
            hrp.CFrame = CFrame.new(tp_pos)
            
            local g = equip_tool(grapple_name)
            if g then
                pcall(function()
                    remote_path:FireServer(unpack(args_tbl))
                end)
            end
            
            task.wait(0.05)
            
            local f = equip_tool(carpet_name)
            if f then
                pcall(function()
                    f:Activate()
                end)
            end
        end
        
        print('🔍 Farm iniciado...')
        while farm_active do
            local highest = find_highest_gen()
            if highest and highest.primaryPart then
                local brain_pos = highest.primaryPart.Position
                local nearest = find_nearest_cf(brain_pos)
                if nearest then
                    local tp_pos = Vector3.new(nearest.X, brain_pos.Y + 2, nearest.Z)
                    tp_sequence(tp_pos)
                end
            end
            task.wait(2)
        end
    end)
end

local function stop_farm()
    farm_active = false
    if farm_thread then
        task.cancel(farm_thread)
        farm_thread = nil
    end
    print('❌ Farm desativado')
end

-- ═══════════════════════════════════════════════════════════
-- SCRIPT 2: ESP VISUAL
-- ═══════════════════════════════════════════════════════════

local function start_esp()
    if esp_active then return end
    esp_active = true
    
    esp_thread = task.spawn(function()
        local plrs = game:GetService("Players")
        local ws = game:GetService("Workspace")
        local lp = plrs.LocalPlayer
        
        local esp_folder = Instance.new("Folder")
        esp_folder.Name = "ESP"
        esp_folder.Parent = lp:WaitForChild("PlayerGui")
        
        local line_folder = Instance.new("Folder")
        line_folder.Name = "ESP_Lines"
        line_folder.Parent = ws
        
        local cur_esp, cur_beam, cur_pet_info, a0, a1
        
        local function parse_mps(txt)
            txt = txt:match("^%s*(.-)%s*$")
            local num, sfx = txt:match("%$([%d%.]+)%s*([kKmMbB]?)%s*/?s?")
            num = tonumber(num) or 0
            if sfx then
                sfx = sfx:lower()
                if sfx == "k" then num = num * 1000
                elseif sfx == "m" then num = num * 1_000_000
                elseif sfx == "b" then num = num * 1_000_000_000 end
            end
            return num
        end
        
        local function create_esp(pet_name, info_txt, mut_txt, target_part)
            if cur_esp then cur_esp:Destroy() end
            
            local bb = Instance.new("BillboardGui")
            bb.Size = UDim2.new(0, 150, 0, 50)
            bb.AlwaysOnTop = true
            bb.Adornee = target_part
            bb.Parent = esp_folder
            bb.StudsOffset = Vector3.new(0, 3, 0)
            
            local lbl = Instance.new("TextLabel")
            lbl.Size = UDim2.new(1, 0, 1, 0)
            lbl.BackgroundTransparency = 1
            lbl.TextColor3 = Color3.fromRGB(255, 255, 255)
            lbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
            lbl.TextStrokeTransparency = 0
            lbl.TextScaled = false
            lbl.TextSize = 10
            lbl.RichText = true
            
            lbl.Text = string.format(
                '<font color="rgb(255,0,0)">%s</font>\n<font color="rgb(0,255,0)">%s</font>\n<font color="rgb(255,255,0)">%s</font>',
                pet_name, info_txt, mut_txt or ""
            )
            
            lbl.Parent = bb
            cur_esp = bb
        end
        
        local function create_beam(target_part)
            local chr = lp.Character
            if not chr or not chr:FindFirstChild("HumanoidRootPart") or not target_part then return end
            
            if cur_beam then
                cur_beam:Destroy()
                if a0 then a0:Destroy() end
                if a1 then a1:Destroy() end
            end
            
            a0 = Instance.new("Attachment")
            a0.Parent = chr.HumanoidRootPart
            
            a1 = Instance.new("Attachment")
            a1.Parent = target_part
            
            local beam = Instance.new("Beam")
            beam.Attachment0 = a0
            beam.Attachment1 = a1
            beam.Width0 = 0.35
            beam.Width1 = 0.35
            beam.FaceCamera = true
            beam.LightInfluence = 0
            beam.Parent = line_folder
            
            cur_beam = beam
            
            task.spawn(function()
                while beam.Parent and esp_active do
                    local color = Color3.fromHSV((tick() % 5) / 5, 1, 1)
                    beam.Color = ColorSequence.new(color)
                    task.wait(0.05)
                end
            end)
        end
        
        local function find_best_pet()
            local plots_folder = ws:FindFirstChild("Plots")
            if not plots_folder then return end
            
            local best_val = -1
            local best_pet = nil
            
            for _, plot in ipairs(plots_folder:GetChildren()) do
                local podiums = plot:FindFirstChild("AnimalPodiums")
                if podiums then
                    for _, podium in ipairs(podiums:GetChildren()) do
                        local spawn_part = podium:FindFirstChild("Base") and podium.Base:FindFirstChild("Spawn")
                        if spawn_part then
                            local attach = spawn_part:FindFirstChild("Attachment")
                            local overhead = attach and attach:FindFirstChild("AnimalOverhead")
                            if overhead then
                                local name_lbl = overhead:FindFirstChild("DisplayName")
                                local gen_lbl = overhead:FindFirstChild("Generation")
                                local mut_lbl = overhead:FindFirstChild("Mutation")
                                if gen_lbl then
                                    local pet_name = name_lbl and name_lbl.Text or "Unknown"
                                    local gen_txt = gen_lbl.Text
                                    local mut_txt = mut_lbl and mut_lbl.Text or ""
                                    local mps = parse_mps(gen_txt)
                                    local gen_num = tonumber(gen_txt:match("%d+")) or 0
                                    local val = mps > 0 and mps or gen_num
                                    
                                    if val > best_val then
                                        best_val = val
                                        best_pet = {name = pet_name, info = gen_txt, mutation = mut_txt, part = spawn_part, value = val}
                                    end
                                end
                            end
                        end
                    end
                end
            end
            
            return best_pet
        end
        
        local function update_esp()
            local pet = find_best_pet()
            if pet then
                local need_update = false
                if not cur_esp then
                    need_update = true
                elseif cur_pet_info then
                    if pet.value > cur_pet_info.value or pet.mutation ~= cur_pet_info.mutation then
                        need_update = true
                    end
                end
                
                if need_update then
                    cur_pet_info = pet
                    create_esp(pet.name, pet.info, pet.mutation, pet.part)
                    create_beam(pet.part)
                end
            else
                if cur_esp then
                    cur_esp:Destroy()
                    cur_esp = nil
                    cur_pet_info = nil
                end
                if cur_beam then
                    cur_beam:Destroy()
                    cur_beam = nil
                end
            end
        end
        
        print('👁️ ESP iniciado...')
        while esp_active do
            update_esp()
            task.wait(0.3)
        end
        
        if esp_folder then esp_folder:Destroy() end
        if line_folder then line_folder:Destroy() end
    end)
end

local function stop_esp()
    esp_active = false
    if esp_thread then
        task.cancel(esp_thread)
        esp_thread = nil
    end
    print('❌ ESP desativado')
end

-- ═══════════════════════════════════════════════════════════
-- CRIAÇÃO DOS BOTÕES
-- ═══════════════════════════════════════════════════════════

local function criar_botao(num, pos_y)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 70, 0, 70)
    btn.Position = UDim2.new(0, 30, 0, pos_y)
    btn.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
    btn.Text = tostring(num)
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.TextSize = 24
    btn.Font = Enum.Font.GothamBold
    btn.AutoButtonColor = false
    btn.Parent = btn_gui

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 12)
    corner.Parent = btn

    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(90, 90, 90)
    stroke.Thickness = 1.5
    stroke.Parent = btn

    local shadow = Instance.new("Frame")
    shadow.Size = UDim2.new(1, 6, 1, 6)
    shadow.Position = UDim2.new(0, -3, 0, -3)
    shadow.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    shadow.BackgroundTransparency = 0.7
    shadow.ZIndex = btn.ZIndex - 1
    shadow.Parent = btn

    local shadow_corner = Instance.new("UICorner")
    shadow_corner.CornerRadius = UDim.new(0, 14)
    shadow_corner.Parent = shadow

    btn.MouseEnter:Connect(function()
        btn.BackgroundColor3 = Color3.fromRGB(55, 55, 55)
    end)

    btn.MouseLeave:Connect(function()
        btn.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
    end)

    btn.MouseButton1Down:Connect(function()
        btn.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    end)

    btn.MouseButton1Up:Connect(function()
        btn.BackgroundColor3 = Color3.fromRGB(55, 55, 55)
        
        if num == 1 then
            if farm_active then
                stop_farm()
                btn.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
            else
                start_farm()
                btn.BackgroundColor3 = Color3.fromRGB(0, 255, 0)
            end
        elseif num == 2 then
            if esp_active then
                stop_esp()
                btn.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
            else
                start_esp()
                btn.BackgroundColor3 = Color3.fromRGB(0, 100, 255)
            end
        elseif num == 3 then
            print("Botão 3 - Reservado para futuras funcionalidades")
        end
    end)
    
    return btn
end

criar_botao(1, 40)
criar_botao(2, 120)
criar_botao(3, 200)

-- ═══════════════════════════════════════════════════════════
-- SOM DE NOTIFICAÇÃO
-- ═══════════════════════════════════════════════════════════

local snd_plr = game:GetService("Players")
local snd_local = snd_plr.LocalPlayer
local snd_gui = snd_local:WaitForChild("PlayerGui")

local audio = Instance.new("Sound")
audio.SoundId = "rbxassetid://18886652611"
audio.Volume = 5
audio.Looped = false
audio.Parent = snd_gui

audio:Play()

-- ═══════════════════════════════════════════════════════════
-- ANTI-RAGDOLL SYSTEM
-- ═══════════════════════════════════════════════════════════

local p = game:GetService("Players").LocalPlayer
local c = p.Character or p.CharacterAdded:Wait()
local h = c:WaitForChild("Humanoid")
local r = c:WaitForChild("HumanoidRootPart")
local rs = game:GetService("RunService")

r.Anchored = false
r.CustomPhysicalProperties = PhysicalProperties.new(100, 0, 0, 100, 100)

local lastPos = r.Position
local lastCF = r.CFrame

local function block()
    p:SetAttribute("RagdollEndTime", nil)
    c:SetAttribute("RagdollEndTime", nil)
end

p.AttributeChanged:Connect(function(a)
    if a == "RagdollEndTime" then block() end
end)

c.AttributeChanged:Connect(function(a)
    if a == "RagdollEndTime" then block() end
end)

h.StateChanged:Connect(function(o, n)
    if n == Enum.HumanoidStateType.Physics or 
       n == Enum.HumanoidStateType.Ragdoll or 
       n == Enum.HumanoidStateType.FallingDown then
        h:ChangeState(Enum.HumanoidStateType.RunningNoPhysics)
    end
end)

for _, m in pairs(c:GetDescendants()) do
    if m:IsA("Motor6D") then
        m:GetPropertyChangedSignal("Enabled"):Connect(function()
            if not m.Enabled then m.Enabled = true end
        end)
    end
    if m:IsA("BasePart") and m ~= r then
        m.CustomPhysicalProperties = PhysicalProperties.new(0.01, 0, 0, 100, 100)
    end
end

c.DescendantAdded:Connect(function(d)
    if d:IsA("BallSocketConstraint") or d:IsA("HingeConstraint") then
        d:Destroy()
    elseif d:IsA("BodyVelocity") or d:IsA("BodyForce") or d:IsA("BodyPosition") or d:IsA("BodyGyro") or d:IsA("BodyThrust") or d:IsA("BodyAngularVelocity") then
        d:Destroy()
    end
end)

local locked = false

rs.Heartbeat:Connect(function()
    for _, d in pairs(c:GetDescendants()) do
        if d:IsA("BodyVelocity") or d:IsA("BodyForce") or d:IsA("BodyPosition") or d:IsA("BodyGyro") then
            d:Destroy()
        end
    end
    
    local moving = h.MoveVector.Magnitude > 0.1 or h.Jump
    if moving then
        lastPos = r.Position
        lastCF = r.CFrame
        locked = false
    end
    
    local vel = r.AssemblyLinearVelocity
    if vel.Magnitude > 50 and not moving and not locked then
        r.CFrame = lastCF
        r.AssemblyLinearVelocity = Vector3.zero
        r.AssemblyAngularVelocity = Vector3.zero
        locked = true
        task.wait(0.1)
        locked = false
    end
end)

r:GetPropertyChangedSignal("CFrame"):Connect(function()
    if locked then
        r.CFrame = lastCF
    end
end)

h:SetStateEnabled(Enum.HumanoidStateType.FallingDown, false)
h:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, false)
h:SetStateEnabled(Enum.HumanoidStateType.Physics, false)

block()

print("═══════════════════════════════════════════════════")
print("✅ Script totalmente carregado!")
print("📱 Compatível com: iOS, PC, Android, Samsung")
print("🚀 Notificações ultra rápidas ativadas!")
print("═══════════════════════════════════════════════════")
