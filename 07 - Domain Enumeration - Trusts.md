- In an AD environment, trust is a relationship between two domains or forests which allows users of one domain or forest to access resources in the other domain or forest.
- Trust can be automatic (parent-child, same forest etc.) or established (forest, external).
- Trusted Domain Objects (TDOs) represent the trust relationships in a domain.

# Trust Direction

**One-way trust** - Unidirectional. Users in the *trusted domain* can *access* resources in the *trusting domain* but the reverse is not true.

![[one-way-trust.png]]

**Two-way trust** - Bi-directional. Users of *both domains* can *access* resources in the *other domain*.

![[two-way-trust.png]]

# Transitivity

**Trust** in AD = _“A accepts authentication from B”_ (i.e., A will allow B’s users to access A’s resources according to permissions).
### Transitive trust

- **Definition:** A trust that _can be chained/extended_ — if **A trusts B** and **B trusts C**, then **A trusts C** automatically.
- **Default use:** All intra-forest trusts (parent↔child, tree-root) are **transitive** and **two-way** by default.
- **Practical effect:** Authentication/identity flow can travel across multiple domains in the forest. Domains form a mesh of trust.
- **Red-team impact:** Compromise in one domain can be leveraged across the forest via chained trusts (easier lateral movement). *Example:* user in `child.domain` can be used to access resources in `parent.domain` if permissions exist, because the trust chain is recognized.

### Nontransitive trust

- **Definition:** A trust that **does not extend** beyond the two domains in the trust. If **A trusts B** and **B trusts C**, **A does NOT automatically trust C**.
- **Default use:** External trusts (between two domains in different forests when no forest trust exists) are **nontransitive** by default. They may be one-way or two-way depending on configuration.
- **Practical effect:** Trust is limited to that explicit pair — you must explicitly create further trusts for other domains.
- **Red-team impact:** Limits lateral movement across the enterprise because the trust boundary stops chains of authentication. An external one-way trust can still allow access in one direction only.

> [!summary]
> - **Transitive + two-way = big risk.** One compromise → potential forest pivot (if authorization exists).
> -  **Nontransitive / one-way = containment.** Limits the scope of compromise; attacker cannot automatically chain through other domains.
> - **Practical test:** Always map trust direction and transitivity during recon — it tells you which accounts/resources you can reach without extra privilege escalation.


> [!note]
> - **Transitive:** trust chains (A→B & B→C ⇒ A→C). Default intra-forest. Usually two-way.
>-  **Nontransitive:** trust stops at the pair. Default for external trusts across forests. Can be one- or two-way.
>-  **Direction matters:** “A trusts B” means **B’s** users can access **A** (if allowed).

# Default/Automatic Trusts

**Parent-child trust** 
- It is created automatically between the new domain and the domain that precedes it in the namespace hierarchy, whenever a new domain is added in a tree. For example, *dollarcorp.moneycorp.local* is a child of *moneycorp.local* 
- This trust is always two-way transitive.

**Tree-root trust**
- It is created automatically between whenever a new domain tree is added to a forest root. 
- This trust is always two-way transitive.

![[Intra-Domain Trust.png]]

# External Trusts

- Between two domains in different forests when forests do not have a trust relationship.
- Can be one-way or two-way and is nontransitive.

![[External-Trusts.png]]

In **External-Trusts** only explicitly specified resources will be shared, e.g.
(If there was a shared folder on *Z* called *Files*, users in *C* will be able to access it only if it was explicitly meant to be shared).

# Forest Trusts

- Between forest root domain. 
- Cannot be extended to a third forest (no implicit trust). 
- Can be one-way or two-way transitive.

![[Forest-Trust.png]]

# Domain Trust Mapping

```powershell
# Get a list of all domain trusts for the current domain 
Get-DomainTrust ## (PowerView)
Get-DomainTrust -Domain us.dollarcorp.moneycorp.local ## (PowerView)

Get-ADTrust ## (ActiveDirectory Module)
Get-ADTrust -Identity us.dollarcorp.moneycorp.local ## (ActiveDirectory Module)


# Get details about the current forest 
Get-Forest ## (PowerView)
Get-Forest -Forest eurocorp.local ## (PowerView)

Get-ADForest ## (ActiveDirectory Module)
Get-ADForest -Identity eurocorp.local ## (ActiveDirectory Module)


# Get all domains in the current forest 
Get-ForestDomain ## (PowerView)
Get-ForestDomain -Forest eurocorp.local ## (PowerView)

(Get-ADForest).Domains ## (ActiveDirectory Module)


# Get all global catalogs for the current forest 
Get-ForestGlobalCatalog ## (PowerView)
Get-ForestGlobalCatalog -Forest eurocorp.local ## (PowerView)

Get-ADForest | select -ExpandProperty GlobalCatalogs ## (ActiveDirectory Module)


# Map trusts of a forest  
Get-ForestTrust ## (PowerView)
Get-ForestTrust -Forest eurocorp.local ## (PowerView)

Get-ADTrust -Filter 'msDS-TrustForestTrustInfo -ne "$null"' ## (ActiveDirectory Module)
```

