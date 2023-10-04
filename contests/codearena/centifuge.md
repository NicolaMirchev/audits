# Medium

## Summary

ERC20 implementation implements EIP712, but has functionallity to change token name, which will result in broken DOMAIN_NAME_SEPARATOR and so invalid signature and impossibility to use Permits

## Details

ERC20 implementation allows change of the name and the symbol. The implementation also covers EIP 712 standard for signing permit functions. The problem is that `name`
of the token is included inside the `DOMAIN_SEPARATOR` of the contract and it is calculated only first time:

```

    function file(bytes32 what, string memory data) external auth {
        if (what == "name") name = data;
        else if (what == "symbol") symbol = data;
        else revert("ERC20/file-unrecognized-param");
        emit File(what, data);
    }
```

If the token name is later changed by the following function:

```

    function file(bytes32 what, string memory data) external auth {
        if (what == "name") name = data;
        else if (what == "symbol") symbol = data;
        else revert("ERC20/file-unrecognized-param");
        emit File(what, data);
    }
```

So if for examle original name of the token was "Test token" and then it has changed to "Better name token", now a signer would use "Bettter name token" for the 'domainName' value in ,the hash struct for domain_separator, which will lead to always failing ecrecover for the signature, even if other data is valid.

## Impact

The impact is not high, because if the name is changed, it could be used the old name for hashing the domainSeparator, but it leads to bad and confusing design and it breaks the standard, because after changing:
`Definition of domainSeparator
domainSeparator = hashStruct(eip712Domain)
where the type of eip712Domain is a struct named EIP712Domain with one or more of the below fields. Protocol designers only need to include the fields that make sense for their signing domain. Unused fields are left out of the struct type.
string name the user readable name of signing domain, i.e. the name of the DApp or the protocol.`

## Recomendations

- Consider removing the option to file the name/symbol of a token, or
- on name change recalculate domainSeparator: (Not a good choice in my opinion)

```
   function file(bytes32 what, string memory data) external auth {
        if (what == "name"){
		name = data;
		// Recalculate DOMAIN_SEPARATOR (Not a good choice)
		}
        else if (what == "symbol") symbol = data;
        else revert("ERC20/file-unrecognized-param");
        emit File(what, data);
    }
```
