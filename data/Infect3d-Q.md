# L-01 UpdateConfig do not prevent `minTenor > maxTenor`

## Vulnerability details
`UpdateConfig::executeUpdateConfig` allow `ISizeAdmin` to modify the market configuration.
While the `Initialize::validateInitializeRiskConfigParams` ensure `minTenor < maxTenor`, this not the case for `UpdateConfig::executeUpdateConfig`


## Impact
Such a missconfiguration would cause multiple functions to revert, especially when `tenor` is checked against `minTenor` and `maxTenor`, as `minTenor < tenor < maxTenor` cannot hold in these conditions.


## Tools Used
Manual review

## Recommended Mitigation Steps
Ensure `minTenor < maxTenor` in `UpdateConfig::executeUpdateConfig`
