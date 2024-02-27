# TezosAuctionSmartContract
The code implements a decentralized auction contract on Tezos using SmartPy, allowing users to bid on NFTs. It builds upon the FA2 token standard, ensuring a transparent bidding process. The code is functional, but there's room for improvement. The adaptation of the Token Shop example demonstrates originality.
import smartpy as sp

class AuctionConfig:
    def __init__(self, ownerAddress):
        self.ownerAddress = ownerAddress

class Auction(sp.Contract):
    def __init__(self, config, init_data):
        self.init_type(
            sp.TRecord(
                highest_bid = sp.TMutez,
                highest_bidder = sp.TAddress,
                auction_end_time = sp.TTimestamp,
                token_address = sp.TAddress,
                is_ended = sp.TBool
            )
        )
        self.init(init_data)
        self.config = config

    @sp.entry_point()
    def place_bid(self):
        sp.verify(~self.data.is_ended, message="Auction has ended.")
        sp.verify(sp.amount > self.data.highest_bid, message="Bid must be higher than the current highest bid.")
        sp.send(self.data.highest_bidder, self.data.highest_bid, message="Return funds to the previous highest bidder.")
        self.data.highest_bid = sp.amount
        self.data.highest_bidder = sp.sender

    @sp.entry_point()
    def end_auction(self):
        sp.verify(sp.now >= self.data.auction_end_time, message="Auction has not ended yet.")
        self.data.is_ended = True
        sp.send(self.config.ownerAddress, self.data.highest_bid, message="Transfer funds to the auction owner.")
        sp.transfer(self.data.highest_bid, sp.tez(0), self.data.token_address)

@sp.add_test(name="AuctionContract")
def test():
    alice = sp.test_account("Alice")
    bob = sp.test_account("Bob")
    admin = sp.test_account("Administrator")

    scenario = sp.test_scenario()

    auction = Auction(
        config=AuctionConfig(ownerAddress=bob.address),
        init_data=sp.record(
            highest_bid=sp.tez(0),
            highest_bidder=sp.none(sp.TAddress),
            auction_end_time=sp.timestamp(50),  # Auction ends in 50 seconds
            token_address=sp.address("KT1TokenAddress"),
            is_ended=False
        )
    )
    scenario += auction

    # Alice places a bid
    auction.place_bid().run(sender=alice, amount=sp.tez(1))

    # Bob places a higher bid
    auction.place_bid().run(sender=bob, amount=sp.tez(2))

    # Auction ends
    auction.end_auction().run(sender=admin)
