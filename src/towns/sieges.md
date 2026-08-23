# Sieges
A town under siege is attacked by a land army that has to be raised, marched to the town and
brought against its gate. How that army is put together and placed is in
[Initialization](./sieges/initialization.md).

A siege also changes what other systems will do:

- while a town is sieged, blocked, boycotted or under pirate attack, visiting the tavern's
  weapons dealer cannot start a
  [criminal investigation](../operations/0052-tavern-interaction.md#weapons-dealer);
- a town that **repels** its attackers hands every population type a satisfaction bonus,
  beggars included, which is the
  [siege beggar satisfaction bug](../bugs/siege-beggar-satisfaction-bonus.md).
