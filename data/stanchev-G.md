GAS-01 validateYieldCurve() tenor validation gas optimization
In validateYieldCurve we can have a gas inefficiency if the tenors are of large amount. We can save gas by putting the checks for the tenors above the checks for the aprs and marketRateMultipliers. This will save some gas in case of out of range tenor.


```javascript
function validateYieldCurve(YieldCurve memory self, uint256 minTenor, uint256 maxTenor) internal pure {
        if (self.tenors.length == 0 || self.aprs.length == 0 || self.marketRateMultipliers.length == 0) {
            revert Errors.NULL_ARRAY();
        }
        if (self.tenors.length != self.aprs.length || self.tenors.length != self.marketRateMultipliers.length) {
            revert Errors.ARRAY_LENGTHS_MISMATCH();
        }

        // validate aprs
        // N/A

        // validate tenors
        uint256 lastTenor = type(uint256).max;
        for (uint256 i = self.tenors.length; i != 0; i--) {
            if (self.tenors[i - 1] >= lastTenor) {
                revert Errors.TENORS_NOT_STRICTLY_INCREASING();
            }
            lastTenor = self.tenors[i - 1];
        }
        //@audit-gas put these above validating the tenors to save gas in case of out of range tenor
        if (self.tenors[0] < minTenor) {
            revert Errors.TENOR_OUT_OF_RANGE(self.tenors[0], minTenor, maxTenor);
        }
        if (self.tenors[self.tenors.length - 1] > maxTenor) {
            revert Errors.TENOR_OUT_OF_RANGE(self.tenors[self.tenors.length - 1], minTenor, maxTenor);
        }

        // validate marketRateMultipliers
        // N/A
    }
```
Recommended mitigation

```javascript
function validateYieldCurve(YieldCurve memory self, uint256 minTenor, uint256 maxTenor) internal pure {
        if (self.tenors.length == 0 || self.aprs.length == 0 || self.marketRateMultipliers.length == 0) {
            revert Errors.NULL_ARRAY();
        }
        if (self.tenors.length != self.aprs.length || self.tenors.length != self.marketRateMultipliers.length) {
            revert Errors.ARRAY_LENGTHS_MISMATCH();
        }
        if (self.tenors[0] < minTenor) {
            revert Errors.TENOR_OUT_OF_RANGE(self.tenors[0], minTenor, maxTenor);
        }
        if (self.tenors[self.tenors.length - 1] > maxTenor) {
            revert Errors.TENOR_OUT_OF_RANGE(self.tenors[self.tenors.length - 1], minTenor, maxTenor);
        }


        // validate aprs
        // N/A

        // validate tenors
        uint256 lastTenor = type(uint256).max;
        for (uint256 i = self.tenors.length; i != 0; i--) {
            if (self.tenors[i - 1] >= lastTenor) {
                revert Errors.TENORS_NOT_STRICTLY_INCREASING();
            }
            lastTenor = self.tenors[i - 1];
        }
        // validate marketRateMultipliers
        // N/A
    }
```