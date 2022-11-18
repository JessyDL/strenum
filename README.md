# strenum
C++20 support for stringifying enums at compile time (with constraints) for *The Big Three* compilers (GCC, CLang, and MSVC). 

## How it works
(Ab)using the compiler injected macro/extension `__FUNCSIG__` or `__PRETTY_FUNCTION__`, at compile time parse the resulting string to extract the enum value, while rejecting entries that do not point to named values.

I don't take responsibity for when this breaks, compiler provided macros and extensions can always be changed, but as of writing (and looking at historical behaviour of these 2 values) they appear to be pretty stable. As of now none of the code goes explicitly against the standard, nor relies on UB (aside from the compiler specific macro/extension usage) and so I foresee this to keep working for some time.

## Limitations
The enum types are currently restricted to anything that satisfies `std::is_integral`, and `std::is_scoped_enum`. The integral limitation isn't really needed, it just lowers the potential oversight of unforeseen issues. Feel free to implement the `std::is_arithmetic` constraint, add tests, and make a PR if you're up for it!

Search depth: As we need to iterate over all potential values that are present in the enum, and as iterating over the entire 2^n bits of the underlying type is too heavy; we limit the search depth by default to `1024`, and offer 2 different iteration techniques (`strenum::sequential_searcher`, and `strenum::bitflag_searcher`). Both of these are tweakable, see the following section for info on how to do so.

Duplicate named values (i.e. multiple enum names on the same `value`) will only fetch the first name it sees. So when the following enum is defined:
```cpp
enum class foo {
    bar,
    tan = bar,
    cos,
    sin,
};
```
The output will be an array of `{ "bar", "cos", "sin" }`, notice the missing `"tan"` (see this [on godbolt](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIM6SuADJ4DJgAcj4ARpjE/gAcpAAOqAqETgwe3r7%2ByanpAiFhkSwxcWaJdpgOGUIETMQEWT5%2BAVU1AnUNBEUR0bEJtvWNzTltwz2hfaUDFQCUtqhexMjsHOYAzKHI3lgA1OZmCAQESQogAPQXxEwA7gB0wIQIXlFeSiuyjAT3aCwXAClMAoFABPAAiQQuCgIxEYPguWAAbhdtrtMNDYfD/jC4QwfPcEEkkocTBoAIJkylmLYMHZefYmDZuZAw/CoJnYKlU7F7HZMEF7fioA4AdisFL2Ur2UQapCp0r29QYBw24JlcoV0rQCnlkulaQYespovBTIllIpzDYCiSTFWe1xvJM4q1BEwLCSBndTLcnK1uK8Dj22IA%2BqF%2BMQWExHAJfcLOWKLYqYTG8Mg%2BQIYZhVEliHs7SdYirlOSACpl7AAJXCSYLMfdxD8ICE2AAinJsOEywBJclBMVmjbJg31RwZtAMbO5/NMLxEPYWbAAcR7tcVdeFIBAsvzLqHI6lqfHman7pneznC676o3UpdFiFqFQ28Ng/NWqlV0dY/Tp%2BneaOngABemChgQewALLkgAGqGrbklWbgABLwT2ABa2B1mYGiSPE77Dns36oEksbMLQtCgnsqBIrExB4PsBAIJgjqYA0yAIEBoGkEqCB4Aoez8QwYAcBBYSYFg6B7F4DD0IKoJLHsfEQUxLG0HgLCEPcWr7h%2BFK6fpFKhBB0ahBAcxJlqk4AbO84ikiYheMCqrqk6%2BIsK%2BsKhE8VCgvGz6cuZenklKkYQNZEFXhyZgAGw/vmIB7A53jAhZD6fqO6DbnmxlUBApJmAArAoJiFW4DCHDxuL3OgMZMOZczBfeppanCBDLCqGjBfuHALLQnCFbwfgcFopAvhwfqWNYjpLCsLGbDwpAEJovULAA1v4or3PEZgAJzxJIZhcBshW7btMVcFw%2BicJIQ0rWNnC8OcGhLStCxwLAMCICgqCenQsTkJQfxJP9cTAAozBnAgqAEKQyLppgABqeCYLcADySSMJwi00LQjbnBAUT3VEoQNKC2O8H8bCCGjsnkyNvBYNGRjiAzcN4HCNS0ecbM5tU85rItxmYP1bPqVENzEKCHhYBTS30R53C9XwBjg8jqMY1jSsyIIIhiOwXABPwgiKCo6hs7oV0GEYKDWNY%2Bh4FE5yQAsJFkTzAC0bJMuCphTZYOF7B7aMbE9IvVGRLgMO4ngtHowRTCUZR6CkaRkaMfhXanBQML0ScDFd7RkV0Iyxzkhfhx0DAl5MxT9HEhcTBneipo0ef1xICwKLNqyd9dHCDaQw2jeNeyqPEMUezFkh7MAyAZhAsIyWtFkQLghAkAcNJcHMvDLQzcwLMxTBYHE5mkBtXCFdtF1mHfhWSBod%2BipI5397dpAeTFZj3FwGiXTFUUGwr5cFFIVK6w9eDjSeiAF6%2B8tDvS%2BhAJAfNkB2TIBQCADRwbKEMCLIQ0NbjDUWsDOgaYBC4LCLQAhqAiH3VIfQMGEMmBQxhqQBhsQ0bzhoXQ7WqDyTEHBo9QIqhqh1HwMNXgxthCiHEFIHW8glBqHuroDY%2BhDDGDtpYB2Tt4Cu1IhkT23s1R%2BysAHUa3dli90bhIyh%2BDCHEL3nCQWvBbg3CSBTPqA07ps3GtgURaCiD5nHpPaes9557EXsQZeq9JpmJsHsdeQSt7AN3q9A%2BR82Kn0oF4jgH8PIbC2vEfa8QNixQOjtV%2BQ97rQNsLA9JCD1r%2BEkPcXaFRJAbA2Bob%2BZgOllP7qHapvjhHwOVh9KASCkAcIwUDX6INGEoGtsAcC0SGBrThpgJECN1bo0xo4vgdB8aUCJmzEmzApZyypt8WmlF7pMw0azUa%2BBOaOG5vdVBAs5bC1FqNcWktpYYBcfLDSniVZMDVijXZWscayD1nIo2shTbKItiANRSzbb%2BxsOLZ2583aGM4F7Ag6AfamOsIHYOgyi4ZCjjHbImdAjR3bjMBueQ04ZGblnfIZEmXJwrvYYuTcy70qpZ0CYPKC5DG6ByyVbdE4dx3osKxBtcmD0gQ9DgY8J5TxnvyIwkSl5rNXkkzeC00mjMPqQY%2B2Tz4bQ6fcB%2BhtCoP12oVbpe1RTv14F/H%2Bf8AFAJAWAiBNThHPQaWMyZIAPlBMBlgwRCg7HUIcZcuZZCyIJp4Wq6ZiyNErOXuwlNjCuHCCTXwgJAihEcF4Kg8RoRhHSLhQbaQ0ikXm1GpbdRNtSXaOxXosaBiswEuMb7LRFgcJPR7gbIYti8GJtofsrEQK3EsNBaLVVwaOD%2BP5skkJ2q%2BRLP1aslekS4n20SfgZJpq95vUySfAY59Rb5NRUUkpZSYoVN6btIZI8Q31PNU0u%2B9qNAPyKoU0UZhRTxEKhsRIotBlqtqea0g4zkNTILQDTBWakTIGJKGJEXBdqhm9MCAgoZVDTw2Vs1YOzNb7NxkcwmxNSYXO1lcmmdM7kegeWsJ5HMI54DebzAJnztbfPun8smMtuNOJBUrBYVBVYKGo3suW9bZGNoUS2lRIBpDoq7Vix2OL9Hu04JiLKenA4XDRmYIi4JwjgmwBYOQy4iLoTwERZcVELjLgAGJETRsoMsIAexuF85ZgLIAqzYFC3jZAwAw78upRAVw0qE512ZSnLl7KhUZbZYUOV6W%2BV8dFVK7LhWq413FSy1uTRSsytrtMXlXcJ191XT479GqyMzxYAoLDSV8P3CIzCSJxq9zbzNdei%2BqKNj3E6bNub83PWfxAN/X%2B/8uCAOAeAwNX6oE/rgW9JDEao0kBjdg%2BNM6M3Jr%2Bt6DI6aS2jUw9hpIuH8OEZjMR0j5HplFsu6W6o5aeZVoCTWyRCiG0SCbYipRradC5F0yOnRhm%2B3GY4KZjF8Sx2Krmi3IloQ7tzrlguuWS6PGyf7mu4ZG6hPbrCd13reHdoDfe0NiAJ7tFno3qN1JV6MmLYKc%2B%2BIpTyl4Q/Tt9VMD9u842r0%2B4QDv5nUOpIEXuFCoDLa7tytYaLWizMOr8XWuFi0WIGkZwkggA%3D)). This is because during compilation `foo::tan` is considered an alias of `foo::bar`, and will be substituted. There is no way of recovering this information.

## Usage & customization points
The only two points you need to interact with is the `strenum::stringify<YOUR_ENUM_TYPE>()` and the specializable `strenum::enum_information`. The first is an `immediate function` that will return you an array of all uniquely named enum values if given an enum type, or if given an enum value will stringify that; while the latter is a customization point (see example section).

If you can't add the `_BEGIN`/`_END` values to the enum, or prefer not to, you can specialize `strenum::enum_information` for your enum class. Note that if you specialize, and set the `END` value in `strenum::enum_information` to be an instance of your enum type, it will be an *inclusive range*, unlike normal ranges, otherwise it will be exclusive range. This is the mathematical difference of `[0,10]` (range of 0 to 10, inclusive) and `[0,10)` (a range of 0 to 9, excluding 10).

By default the search iterations is limited to `1024`, this means if the difference between the first and last enum value is larger than that, you'll either have to specialize `strenum::enum_information` for your type, or globally override the default value by defining `STRENUM_MAX_SEARCH_SIZE` with a higher value.

Lastly the search pattern. There are 2 provided search patterns `strenum::sequential_searcher` and `strenum::bitflag_searcher`. Both will search from `_BEGIN` to `_END`, but have a different approach.
- `sequential_searcher`: iterates over the range by adding the lowest integral increment for the underlying type.
- `bitflag_searcher`: iterates over the range by jumping per bit value instead (so an 8bit type will have 8 iterations, one for every bit). Combinatorial values are not searched for. For example if there is a value at 0x3, which would be both first and second bit set, it would be skipped.

You can provide your own searcher, as long as it satisfies the following API:
```cpp
struct custom_searcher {
  template<typename T, auto Begin, auto End, auto GetEnumName /* optimized version of stringifying enums*/>
  consteval auto operator() const noexcept -> std::array<std::string_view, /* size must be calculated internally */>
};
```
See `strenum::sequential_searcher`, or `strenum::bitflag_searcher` for example implementations.

## Examples

### compile time stringify an enum ([godbolt](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIM6SuADJ4DJgAcj4ARpjE/lykAA6oCoRODB7evv5JKWkCIWGRLDFxZgl2mA7pQgRMxASZPn4BldUCtfUEhRHRsfG2dQ1N2a1D3aG9Jf3lAJS2qF7EyOwc5gDMocjeWADU5mYIBASJCiAA9OfETADuAHTAhAheUV5Ky7KMBHdoLOcAUpgFAoAJ4AESC5wUBGIjB85ywADdzlsdpgoTC4X9obCGD47ghEokDiYNABBUlkrG7bZMYG7fiofYAdis5N2HN2UXqpF2dQYvLQCl5qQFlJMzLBJnWbIp5OYbAUiSYK12OOpEtlHIImBYiQMOulbml2EpHJxXgcuyxAH1QvxiCwmI4BEbGSaWVrOW9QsBdkJsGSAEpuAAS2CD%2B3WYLVmLxLBAICUAEcvF88GIbUp6sgELFpV7zXVHMgaQJoZhVIliLsmF4iLsLNgAOIASXCnM5moZqFQie5NYlUplZs50OdeFLaAYFarNbrDew4Rjna7rJ7faToRZw8Lu0usYnU/LOrnarwAC9MDaCLsALJkgAaNoDwbDL9bAC1sJ7dmYNJIAAcO4Fvu5y7KgiQuswtC0CCEGIrExB4HsBB5mqmA5gg55XryaF4AouwEQwYAcLeYSYFg6C7F4DD0PSIKLLsCCEHy6G0HgLCEHco5DgW4qSuK5KhLeTqhBAsyeqO06ztWtb1kyiJiGmhHSjG6rxomOK%2BngVAgm6vYmhJ/HsoeJY2nS7wEBASneECJgAKwWBojkxmpan7GYZgDgcswmWSHKwgQSwMLsLkjuSQ4cPMtCcA5vB%2BBwWikH2HDGpY1hqosyyYJ56w8KQBCaNF8wANb%2BMydyAWYACcgGSOU6wOTVNUAGxcAksUcJICXFSlnC8GcGiFcV8xwLAMCICgqB6nQsTkJQvyJHNcTAAozCnAgqAEKQSKTpgABqeCYDcADyiSMJwBU0LQOrEGcEBRH1UShPUIJXbwvxsIIp10e9SW8FgTpGOIAO7XgsLVIhZxg5WVT1qsBUiZgXXJRxUTXMQIIeFgH2FchCbcNFfAGGtR0nedl1EzIggiGI7BcAE/CCIoKjqGDuidYYxjWNY%2Bh4FEZyQPMkHQTDAC00LoGppgZZY/67OLp3rINKNVNBLgMO4njNHowSTMUpR6MkqTQSMfgJCb%2BQMD0hv9BUavtAwnTDDr2QO/Y0EuxMRR9HEFTjObejjg0tt%2BxI8wKNlKwR/ocW9WDqW7KogGteLrWSLswDIKWEAwrRpWSRAuCECQeVcLMvBFQDszzHmTBYHEEmkOVXAOVV7VeWYDmSBoXnMpIbVx91vAJq1Zh3FwGgda1zL5Q5XDMgvpCJclqWDSAw3V1oY2TRASBw8gClkBQED1GtyiGCjQhbTciUFUtdATgIl9hLQN%2BoHffWP/Qq3rUwm1tqkB/rEU69YP5f2pofMkxA1oDUCKoKotR8CJV4MzYQohxBSBpvIJQag%2Bq6HWPobmKBeaWH5oLeAIsoLpAllLGWZCLD/kGtHBmgwUGv2vrfe%2BVdYSI14Dca4iQPoxXjivPqqVsCIKPkQGsKc04ZyzjnXYediAFyLulKw5Ddgl1keXSuI0a510wo3SgoiR6kATOsSqgE6qAXWGYVq9VqqD3EYneBQ1DE7zKv4SQdwapmHqusdYGhx5mEkMEgIXUVZuLXvA7exNxpQD3kgEBJ9FozWWr/FABgjA3jUQwUqu1MCIn2uTM6F0eF8DoHdB6T0wYvWYFjPGX0vi/Tgn1IG3NQbJXwJDRw0M%2BqHwRnjZGqNeDo0xtjDA/D8acRESTJgZNjoVKptdWQdMsFM1kKzfBHMQBENyTzOWNh0ZC2bqLWhnBJYEGltGWWWimEaEVsrVWnt0ia21lkC2gQtZh2mP7XIpt0hB0tnkaC/yjYe3VjUQObsfltC9uMSF9tBhdFBWi0OBtw4VwWEsGOuKurxVibwJO8j06Z1pEYFR%2BdClF10WXDYFcq6jWMQ3fozdyoRLuD3RmDke41QcqE2qzJh49UsSAcek9p5cFnvPRey9V6ko8ZvLxiSUkgGGbIhaZ9YEKE4e/bhLTMlP2ggaiBSrgEmuyVS4A%2BSC5Wtmr/MBwgjVQOkTAuBHBeCH2QaEeB6DNkM2kOg3Z7Nkqc2IUYUhJyKHnOoWLa59D7mMOYXinKwdbmhHNW6gqmJZmCIAQsolCc4kcCkfDPR5LFG2ppQUwuKjNF8x0fgPRTKDEJNrqQeupjm5dXFVYmxdiHFOKAuEmqJL%2BretsKqztPivI8o0D3bu1jmRmGZIBBy6xALDxiZa9eard7wCSfvaaTr5qnzSSAREyAiQ2kRFwGqNoDRAgIDaVQGdimlJWOUymVSbq1MoPU5KjS3rGpYN9Ag7T/q9N1N01YvSIYwsGbDaRIzqZjL6pMt6OMEO8PmUTeYVBSYKF/ZUvGgbMHBpwWGghIBpBHJjY8uNVCUo0PLJwDE6AmPWAVucU6Zh9xgmXNgCwchmz7k/HgfczZ4LnGbAAMX3KdZQAAVEArY3BKf42pkAQZsDadusgYAbyYXOAgK4DF%2BtfYAuNuCkF8K7PAoKNi2z0KnbewxYi2FXQUWApDo0RzAdfOuahZHVhscS2TrJYolgCgb27AfTVO4L7oQqIZYOMw%2BUO2spbgc9YdxglFeKyVsVo9JUTynjPOebcFUJH3Sqreo1SAnoPmh7Vp9z76qvoaz%2BVSf7PwYDmvr39rX9BvXepLz7nSvvfZ%2BtJLqLV42gXq%2BBvqs2oJwUGiQIadl4PDToHIjGHl8zOaxy5HGOBcZ4/LZKUd8VsKltmnrS3qb5rxoW4RhHh7Eoa%2BW9rZcP2ZziwlpLKWZtpYgE27RGX9EsqMWViV1iqrDscc48d0XGuHvnX4ue48WoNUkGOgCDld2luVdOudw8zDk6nfD7xpBEL3Q%2BZIIAA%3D%3D))
Following example showcases an enum being stringified, and accessible as a contiguous array of `std::string_view`'s.
```cpp
enum class foo {
    bar, tan, cos, sin,
};

namespace strenum {
  template<>
  struct enum_information<foo> {
    using SEARCHER = strenum::sequential_searcher;
    static constexpr auto BEGIN      { foo::bar };
    static constexpr auto END        { foo::sin };
    // static constexpr size_t MAX_SEARCH_SIZE { 2048 }; // optionally override the search size, this isn't needed unless you hit the limit.
  };
}

int main() {
  constexpr auto values = strenum::stringify<foo>();
  static_assert(values[0] == "bar");
  return 0;
}
```

### compile time stringify a single enum value ([godbolt](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIM6SuADJ4DJgAcj4ARpjEElykAA6oCoRODB7evv5JKWkCIWGRLDFxXAl2mA7pQgRMxASZPn4BldUCtfUEhRHRsfG2dQ1N2a1D3aG9Jf3lAJS2qF7EyOwc5gDMocjeWADU5mYIBASJCiAA9OfETADuAHTAhAheUV5Ky7KMBHdoLOcAUpgFAoAJ4AESC5wUBGIjB85ywADdzlsdpgoTC4X9obCGD47ghEokDiYNABBUlkrG7bZMYG7fiofYAdis5N2uyi9VIuzqDB5aAUPNS/MpJmZYJM6zZFPJoQIuxYTFCEFmLJlHLQDGhmFUiWIuyYXiIDNQqC5xAA%2Bjj9uswbscViQCAcaFHlQQVK3IznRapdhVVKNQ66o5kJa6e8CNQzRbrTDbZK7SH0M7XUZLYi8Jgbur9mYzH6CyywbMg5SObCCEsGLsNOXyeKwRx5rROABWXh%2BDhaUioThuazWB2LZaYfPrHikAiaFvzADW/mZdwAHGYAJwryRmLjrdvr9cANnK%2Bk4ki7s77nF4Zw009n8zgsBgiBQqBYiTosXIlF%2Bn/ocTAAozCnAgqAEKQSJ4CsABq2Y3AA8okjCcFONC0AQsRnBAUSXlEoT1CCqG8L8bCCAhDC0ERPa8FgSpGOINGQXgsLVIiQKXrqVTGqsU7ypgbZMbQeBRNcxAgh4WDEdOxB4CwxHzFQBhAXBOZISh3C8PwggiGI7BcAE2nyEoaiXroCQGEYKBDpY%2BgiWckDzKgiSOAIZwcAAtNC6BSmCpiWNYZgaLsHkIesN4CVUrl%2BBArgjH4CTBJMxSlHoySpNF8VpXk0U9ClMy2JF7QMJ0wyeM0ehtNFpUTEUfRlIMXRZRU4x5fVEjzAoo4rB1p4cJ2pDdr2/YcLsqgroeHmHpIuzAMgyC7BAMJeAw85qhAuCECQE5cLMvAzjRszzAgmBMFgcSqqQi5cO2q7HgWZjtpIGgFsykhHn156kPJh5mHcXAaOUh7MpO7ZcMyYODZeI03iAd4HVoj4vhASBccgxokD%2BED1EByiGAJQhgTc3ZTn%2BdBMNFeNhLQhOoMTl5kwBIBASBChgRBjOxAhxq0/TmmBKoVRksQQHXgLVS1Pg3ZabIuniFIMiCIoKjqExujrPohjGDZNjCVEDmXc50XuV5BA%2BXa/lWJYQU3t1%2BmDFLVME0TJP7bCvG8Dc1yJApfUDUNvAjdggvo0QBrjZN02zfNi3Lat62DgFtm7JtYc7Xt96Hcdp3nZQrZnrw8nrMuK6biu6xmIeW5ru9UNMTDthw5niMLv4kh3OuZhbus6waL9ZiSD3ASCeFdfDWLCMtqQT5QMjSCc2QFAQAvKCWcAlpx/OkGYFmsHweprt8HQmHENhuFMfhzDidJpFfBRVGXnRWuMb2%2BCsY47Hub2aM8dJ/GCb2PWYkJIYA9jJOSvslJMBUvvZCh8jJy30oZWQytTJqxABrNe1kk663svAJyLl0gm28r5S2gVgqhVHlVdILgGDuHKtkRKdC2rTAaulfIGQGEJVyBldILDUoVCKtVcYzVCr2GEV0fhBVoRNS4ZVVqyV2q7QWEsHqyjBL%2B2hpwMaE0pozVpEYWOxAVprUWqnbaGxdr7QfNnM6/RLqLkHncJ6Bl2xPXXO2PuG5mSfULiAX6/1AZcGBqDcGkMA5Xg4LDeGD5p5zxAD/MOWMcYKCdjTF2N93z/gpukNJvMIkrwMevTepAF7c2EBk/maNhaiyieLZAktQhiwQaIeW0gjKoNVr2cymsrJkNsnrA2BDjacFNubPyOsbYqLHPIx2%2BN0l00PpiMBXsmA%2B00vnfqF566cGDtxNOEc9E0jXkYkxCcdY8nMQaSxGdJ5HVICdOxF1NlfSLiXMuFcq6SBruuMegcxa3mbnOK6/g/qeKeo9YuzIzDMhXO2dYK4%2BqjwiQ3O5cT4AzxRm%2BD8X5F6/iybikAiJkBEkzFwdcloDCYWhJaVQ01t670wKpRCcDpLoRPmfPCBFr781vuRSi1FX6YHosAF%2BtEWJRTwJ/TiIdf783/peIBhFJKrF7DCCBGy%2BDKQUMyg%2BbLZatP0u0lBJkuk6BANILB/TcH63wX2QhblOAYlTNaoKuxzgITMO6sE4QwTYAsHIAA4u6gAWngd1gaQQRoAGLuoQsoAAKiAAAkm4WNHrE0gAAErYHTRhZAwAIriJobFOhoikp1VYdlXhAhRHsNyooqtgji0dBEXI5tkrW2SMbQIxqZUsjcJkQ0KRZROp216ho7Z49Rp0pmiwBQxLdiInJXcKlQIFQbXwGnG51is4gp7ncHuR7j0nt8d9fxf0AZAxBjdMJCQUUAqbmizFqNZVJKXikvJlTSYEqpbk%2BZ%2BSGa/v6MS0ly6KVrppbO0pwHiDlMA1UkONT3K8DRo06WithCGokMapWpqzI5CtRcwZdqjZEKdd5bBVsLBTK6qo%2B23lQhfsWdJZZ0lVnrJ4JszROyOB7NDttWdioF0LXA6uim67FqJ2o5crdFizCTluTYs9bzVwfMrtXAevyH11MBXc1uA87gg1%2BgebckhvlBUkO2JFU7/l1P031MwtnIm7pbqQdip8aGSCAA%3D%3D%3D))
Here we stringify only a single value of the enum, note the abscence of the customization point as it's unneeded.
The return type of this function is a unique type that holds the string as a NTTP.
```cpp
enum class foo {
  bar, tan, cos, sin,
};

int main() {
   // note: the return type of foobar_str is a type containing the string as NTTP argument.
  constexpr auto foobar_str = strenum::stringify<foo::bar>();
  static_assert(foobar_str == std::string_view { "bar" });
  return 0;
}
```

# Licence

See the [LICENSE](LICENSE) file provided.