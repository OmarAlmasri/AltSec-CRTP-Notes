# What are the external trusts for the current domain?

```powershell
Get-DomainTrust | ?{$_.TrustAttributes -eq "FILTER_SIDS"}
```

# Identify external trusts of dollarcorp domain. Can you enumerate trusts for a trusting forest?

Yes we can, because there is a trusting relationship

```powershell
Get-ForestDomain -Forest eurocorp.local | %{Get-DomainTrust -Domain $_.Name}
```

We can enumerate users, groups, or get a full `BloodHound` database from the `eurocorp.local`, but not for `eu.eurocorp.local`

**Why can't we access `eu.eurocorp.local`?** 
Because of **Transitivity**, there is a trust between `dollarcorp.local` & `eurocorp.local` but not between `dollarcorp.local` & `eu.eurocorp.local`

**Then how can we move to `eu.eurocorp.local` ?**
We can do so if we extracted credentials from `eurocorp.local`, because we'll be using the credentials to authenticate, not Trust.