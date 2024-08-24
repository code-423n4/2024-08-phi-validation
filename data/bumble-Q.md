# [NC-1] Wrong natspec, describing `royaltyMintSchedule` when it is not used.
/// @param royaltyMintSchedule Every nth token will go to the royalty recipient.
/// @param royaltyBPS The royalty amount in basis points for secondary sales.
/// @param royaltyRecipient The address that will receive the royalty payments.
    struct RoyaltyConfiguration {
        uint32 royaltyBPS;
        address royaltyRecipient;
    }

