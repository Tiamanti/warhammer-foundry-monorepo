## Warhammer Foundry Monorepo
Facilitates development of Warhammer Foundry modules for Foundry VTT.   
Types for Foundry VTT: https://github.com/League-of-Foundry-Developers/foundry-vtt-types  
* Warhammer Library (shared between lower modules): https://github.com/Tiamanti/WarhammerLibrary-FVTT  
  * Warhammer Fantasy Roleplay 4e: https://github.com/Tiamanti/WFRP4e-FoundryVTT
  * Warhammer 40k: https://github.com/Tiamanti/ImpMal-FoundryVTT
  * Warhammer The Old World: https://github.com/Tiamanti/OldWorld-FoundryVTT

As is with Foundry VTT environment, the lower modules should assume that higher level modules are loaded.
So Warhammer Fantasy Roleplay 4e should have access to Warhammer Library but not vice versa or Warhammer 40k.
