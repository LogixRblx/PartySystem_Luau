PartySystem.lua

--// NOTICE: Hey fellow dev, if you´re looking for my game here is the correct link: https://www.roblox.com/games/82219685274190/Wraithwood For HiddenDevs :)

--// Types //--

export type PartyData = {
	Leader: Player,
	Members: { Player },
	MaxMembers: number,
	PartyID: string,
	ReadyStates: { [Player]: boolean },
}

export type PartyService = {
	ActiveParties: { [string]: PartyData },
	PlayerToParty: { [Player]: PartyData },

	CreateParty: (self: PartyService, player: Player) -> PartyData?,
	DisbandParty: (self: PartyService, party: PartyData) -> (),
	LeaveParty: (self: PartyService, player: Player) -> (),
	AddPlayerToParty: (self: PartyService, party: PartyData, player: Player) -> boolean,
	RemovePlayerFromParty: (self: PartyService, party: PartyData, player: Player) -> (),
	SendPartyUpdate: (self: PartyService, party: PartyData) -> (),
	AreAllReady: (self: PartyService, party: PartyData) -> boolean,
	SetReady: (self: PartyService, player: Player, ready: boolean?) -> (),
}

local Party: PartyService = {}

--// Variables //--

local TeleportService = game:GetService("TeleportService")

local PartyRemotes = game.ReplicatedStorage.PartyService.Events

local PlaceID = "96293129996863"

--// Tables //--

Party.ActiveParties = {}
Party.PlayerToParty = {
	--[[
	[PlayerA] = Party1,
	[PlayerB] = Party1,
	[PlayerC] = Party2
	]]
}

--// Functions for the parties //--

--// Helpers //--

function Party:SendPartyUpdate(party: PartyData)
	for _, member in ipairs(party.Members) do
		PartyRemotes.PartyUpdate:FireClient(
			member,
			party.Members,
			party.Leader
		)
	end
end

--// Readying up system //--

function Party:AreAllReady(party: PartyData)
	for _, member in party.Members do
		if not party.ReadyStates[member] then
			return false
		end
	end
	return true
end

function Party:SetReady(player: Player, ready: boolean)
	local party = self.PlayerToParty[player]
	if not party then return end

	party.ReadyStates[player] = ready
	self:SendPartyUpdate(party)
	
	print("Player",player.Name,"set ready to state",tostring(ready))
	
	if self:AreAllReady(party) then
		print("Party ready — start match")
		local TpOptions = Instance.new("TeleportOptions")
		TpOptions:SetTeleportData({
			PartyID = party.PartyID,
			Members = party.Members,
			Leader = party.Leader.UserId,
		})
		
		local accessCode = TeleportService:ReserveServer(PlaceID)
		TpOptions.ReservedServerAccessCode = accessCode
		
		TpOptions:SetTeleportData({
			PartyID = party.PartyID,
			MemberIds = table.map(party.Members, function(p)
				return p.UserId
			end),
			LeaderId = party.Leader.UserId,
		})
		
		TeleportService:TeleportAsync(
			PlaceID,
			party.Members,
			TpOptions
		)
	end
end

--// Party Functions //--

function Party:CreateParty(player: Player)
	if self.PlayerToParty[player] then
		return
	end
	
	local newParty = {
		Leader = player,
		Members = { player },
		MaxMembers = 4,
		ReadyStates = {},
		PartyID = game:GetService("HttpService"):GenerateGUID(false),
	}
	print(newParty)
	self.ActiveParties[newParty.PartyID] = newParty
	self.PlayerToParty[player] = newParty
	
	self:SendPartyUpdate(newParty)
end

function Party:DisbandParty(party)
	for _, player in ipairs(party.Members) do
		self.PlayerToParty[player] = nil
		self:CreateParty(player)
	end

	self.ActiveParties[party.PartyID] = nil
end

function Party:RemovePlayerFromParty(party: PartyData, player: Player)
	for i, member in party.Members do
		if member == player then
			table.remove(party.Members, i)
			break
		end
	end
	
	self:SetReady(player, nil)
	self.PlayerToParty[player] = nil
	self:SendPartyUpdate(party)
	self:CreateParty(player)
end

function Party:LeaveParty(player: Player)
	local party = self.PlayerToParty[player]
	if not party then return end

	if party.Leader == player then
		for _, member in ipairs(party.Members) do
			PartyRemotes.PartyUpdate:FireClient(member, nil, nil)
			self.PlayerToParty[member] = nil
			self:CreateParty(member)
		end

		self.ActiveParties[party.PartyID] = nil
	else
		self:RemovePlayerFromParty(party, player) --// TODO: Replication error here!
	end
end

function Party:AddPlayerToParty(party: PartyData, player: Player): boolean
	if self.PlayerToParty[player] then
		self:LeaveParty(player)
	end

	if #party.Members >= party.MaxMembers then
		return false
	end
	
	self:SetReady(player, false)
	
	table.insert(party.Members, player)
	self.PlayerToParty[player] = party

	self:SendPartyUpdate(party)
	return true
end

--// Remotes //--

local CreateParty : RemoteEvent = PartyRemotes.CreateParty

CreateParty.OnServerEvent:Connect(function(Player)
	Party:CreateParty(Player)
end)

local LeaveParty : RemoteEvent = PartyRemotes.LeaveParty

LeaveParty.OnServerEvent:Connect(function(Player : Player)
	Party:LeaveParty(Player)
end)

local InvitePlayer : RemoteEvent = PartyRemotes.InvitePlayer

InvitePlayer.OnServerEvent:Connect(function(Player : Player, InvitedPlayer : Player)
	if not Party.PlayerToParty[Player] then warn("[Party-System]: Player tried to invite but is not in a lobby!") return false, "You´re not in a lobby!" end
	if Party.PlayerToParty[Player] == Party.PlayerToParty[InvitedPlayer] then warn("[Party-System]: Tried to invite player that is already in the lobby!") return false, "Can´t invite player that has already joined the lobby!" end
	PartyRemotes.InvitePlayer:FireClient(InvitedPlayer, Player)
	return true, "Player invited!"
end)

local AcceptInvite : RemoteEvent = PartyRemotes.AcceptInvite

AcceptInvite.OnServerEvent:Connect(function(Player : Player, SentFromPlayer : Player)
	local DesiredParty : PartyData = Party.PlayerToParty[SentFromPlayer]
	if not DesiredParty then warn("[Party-Service]: Party does not exist anymore!") return false, "Party does not exist anymore!" end
	Party:AddPlayerToParty(DesiredParty, Player)
	return true, "Joined party!"
end)

local SetReady : RemoteEvent = PartyRemotes.SetReady

SetReady.OnServerEvent:Connect(function(Player : Player, State : boolean)
	if type(State) ~= "boolean" then return end
	Party:SetReady(Player, State)
end)

return Party
