#===============================================================================
#Timer Based Safari Zone by Hobbs
#Version 1.0.0 01/13/2024
#===============================================================================
# Install instructions
#
# replace existing script in the safari zone script with this script
# adjust timer in settings line 93 as seen below:
#
# The number of seconds a Bug-Catching Contest lasts for (0=infinite).
# BUG_CONTEST_TIME        (Time is set here-->) = 20 * 60   # 20 minutes
#===============================================================================
#
#===============================================================================
class SafariState
  attr_accessor :ballcount
  attr_accessor :captures
  attr_accessor :decision
  attr_accessor :timer_start
  
  TIME_ALLOWED = Settings::BUG_CONTEST_TIME	# In seconds

  def initialize
    @start      = nil
    @ballcount  = 0
    @captures   = 0
    @inProgress = false
    @steps      = 0
    @decision   = 0
  end

  def expired?
    return false if !undecided?
    return false if TIME_ALLOWED <= 0
    return System.uptime - timer_start >= TIME_ALLOWED
  end

  def pbReceptionMap
    return @inProgress ? @start[0] : 0
  end

  def inProgress?
    return @inProgress
  end

  def undecided?
    return (@inProgress && @decision == 0)
  end

  def decided?
    return (@inProgress && @decision != 0) || @ended
  end

  def pbGoToStart
    if $scene.is_a?(Scene_Map)
      pbFadeOutIn do
        $game_temp.player_transferring   = true
        $game_temp.transition_processing = true
        $game_temp.player_new_map_id    = @start[0]
        $game_temp.player_new_x         = @start[1]
        $game_temp.player_new_y         = @start[2]
        $game_temp.player_new_direction = 2
        pbDismountBike
        $scene.transfer_player
      end
    end
  end

  def pbStart(ballcount)
    @start      = [$game_map.map_id, $game_player.x, $game_player.y, $game_player.direction]
    @ballcount  = ballcount
    @inProgress = true
    @timer_start = System.uptime
  end

  def pbEnd
    @start      = nil
    @ballcount  = 0
    @captures   = 0
    @inProgress = false
    @decision   = 0
    $game_map.need_refresh = true
  end
end

#===============================================================================
#
#===============================================================================
class TimerDisplay # :nodoc:
  attr_accessor :start_time

  def initialize(start_time, max_time)
    @timer = Window_AdvancedTextPokemon.newWithSize("", Graphics.width - 120, 0, 120, 64)
    @timer.z = 99999
    @start_time = start_time
    @max_time = max_time
    @display_time = nil
  end

  def dispose
    @timer.dispose
  end

  def disposed?
    @timer.disposed?
  end

  def update
    time_left = @max_time - (System.uptime - @start_time).to_i
    time_left = 0 if time_left < 0
    if @display_time != time_left
      @display_time = time_left
      min = @display_time / 60
      sec = @display_time % 60
      @timer.text = _ISPRINTF("<ac>{1:02d}:{2:02d}", min, sec)
    end
  end
end

#===============================================================================
#
#===============================================================================
def pbInSafari?
  if pbSafariState.inProgress?
    # Reception map is handled separately from safari map since the reception
    # map can be outdoors, with its own grassy patches.
    reception = pbSafariState.pbReceptionMap
    return true if $game_map.map_id == reception
    return true if $game_map.metadata&.safari_map
  end
  return false
end

def pbSafariState
  $PokemonGlobal.safariState = SafariState.new if !$PokemonGlobal.safariState
  return $PokemonGlobal.safariState
end

#===============================================================================
#
#===============================================================================
EventHandlers.add(:on_map_or_spriteset_change, :show_safari_game_timer,
  proc { |scene, _map_changed|
    next if !pbInSafari? || pbSafariState.decision != 0 || SafariState::TIME_ALLOWED == 0
    scene.spriteset.addUserSprite(
      TimerDisplay.new(pbSafariState.timer_start, SafariState::TIME_ALLOWED)
    )
  }
)

EventHandlers.add(:on_frame_update, :safari_game_counter,
  proc {
    next if !pbSafariState.expired?
    next if $game_player.move_route_forcing || pbMapInterpreterRunning? ||
            $game_temp.message_window_showing
    pbMessage(_INTL("ANNOUNCER: BEEEEEP!"))
    pbMessage(_INTL("Time's up!"))
    pbSafariState.decision = 1
    pbSafariState.pbGoToStart
  }
)

#===============================================================================
#
#===============================================================================
EventHandlers.add(:on_calling_wild_battle, :safari_battle,
  proc { |pkmn, handled|
    # handled is an array: [nil]. If [true] or [false], the battle has already
    # been overridden (the boolean is its outcome), so don't do anything that
    # would override it again
    next if !handled[0].nil?
    next if !pbInSafari?
    handled[0] = pbSafariBattle(pkmn)
  }
)

def pbSafariBattle(pkmn, level = 1)
  # Generate a wild Pokémon based on the species and level
  pkmn = pbGenerateWildPokemon(pkmn, level) if !pkmn.is_a?(Pokemon)
  foeParty = [pkmn]
  # Calculate who the trainer is
  playerTrainer = $player
  # Create the battle scene (the visual side of it)
  scene = BattleCreationHelperMethods.create_battle_scene
  # Create the battle class (the mechanics side of it)
  battle = SafariBattle.new(scene, playerTrainer, foeParty)
  battle.ballCount = pbSafariState.ballcount
  BattleCreationHelperMethods.prepare_battle(battle)
  # Perform the battle itself
  decision = 0
  pbBattleAnimation(pbGetWildBattleBGM(foeParty), 0, foeParty) do
    pbSceneStandby { decision = battle.pbStartBattle }
  end
  Input.update
  # Update Safari game data based on result of battle
  pbSafariState.ballcount = battle.ballCount
  if pbSafariState.ballcount <= 0
    if decision != 2   # Last Safari Ball was used to catch the wild Pokémon
      pbMessage(_INTL("Announcer: You're out of Safari Balls! Game over!"))
    end
    pbSafariState.decision = 1
    pbSafariState.pbGoToStart
  end
  # Save the result of the battle in Game Variable 1
  #    0 - Undecided or aborted
  #    2 - Player ran out of Safari Balls
  #    3 - Player or wild Pokémon ran from battle, or player forfeited the match
  #    4 - Wild Pokémon was caught
  if decision == 4
    $stats.safari_pokemon_caught += 1
    pbSafariState.captures += 1
    $stats.most_captures_per_safari_game = [$stats.most_captures_per_safari_game, pbSafariState.captures].max
  end
  pbSet(1, decision)
  # Used by the Poké Radar to update/break the chain
  EventHandlers.trigger(:on_wild_battle_end, pkmn.species_data.id, pkmn.level, decision)
  # Return the outcome of the battle
  return decision
end

#===============================================================================
#
#===============================================================================
class PokemonPauseMenu
  alias __safari_pbShowInfo pbShowInfo unless method_defined?(:__safari_pbShowInfo)

  def pbShowInfo
    __safari_pbShowInfo
    return if !pbInSafari?
    if Settings::SAFARI_STEPS <= 0
      @scene.pbShowInfo(_INTL("Balls: {1}", pbSafariState.ballcount))
    else
      @scene.pbShowInfo(_INTL("Caught: None/nBalls: {1}", pbSafariState.ballcount))
    end
  end
end

MenuHandlers.add(:pause_menu, :quit_safari_game, {
  "name"      => _INTL("Quit"),
  "order"     => 60,
  "condition" => proc { next pbInSafari? },
  "effect"    => proc { |menu|
    menu.pbHideMenu
    if pbConfirmMessage(_INTL("Would you like to leave the Safari Game right now?"))
      menu.pbEndScene
      pbSafariState.decision = 1
      pbSafariState.pbGoToStart
      next true
    end
    menu.pbRefresh
    menu.pbShowMenu
    next false
  }
})
