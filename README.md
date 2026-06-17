# base123xx
0x9e4E1f2B064a489E48EB9b427fD0f742bdEb9e4C
import time
from collections import defaultdict, deque
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

WINDOW_BLOCKS = 20

EARLY_MAX_HOLDERS = 20
GROWTH_MIN_HOLDERS = 100

SCORE_THRESHOLD = 3

ZERO = "0x0000000000000000000000000000000000000000"


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


def main():

    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise RuntimeError("Cannot connect")

    print("Connected to Base")
    print("Detecting early holder-growth wallets...\n")

    last_block = w3.eth.block_number

    token_holders = defaultdict(set)

    token_history = defaultdict(
        lambda: deque(maxlen=5)
    )

    early_wallets = defaultdict(set)

    wallet_scores = defaultdict(int)

    while True:

        try:

            current_block = w3.eth.block_number

            if current_block >= last_block + WINDOW_BLOCKS:

                logs = w3.eth.get_logs({
                    "fromBlock": current_block - WINDOW_BLOCKS,
                    "toBlock": current_block,
                    "topics": [TRANSFER_TOPIC]
                })


                current_wallets = defaultdict(set)


                for log in logs:

                    token = log["address"]

                    sender = decode_address(
                        log["topics"][1]
                    )

                    receiver = decode_address(
                        log["topics"][2]
                    )


                    if receiver != ZERO:

                        token_holders[token].add(
                            receiver
                        )

                        current_wallets[token].add(
                            receiver
                        )


                for token, holders in token_holders.items():

                    count = len(holders)

                    history = token_history[token]


                    if len(history) > 0:

                        previous = history[-1]


                        if (
                            previous <= EARLY_MAX_HOLDERS
                            and count >= GROWTH_MIN_HOLDERS
                        ):

                            for wallet in early_wallets[token]:

                                wallet_scores[wallet] += 1


                    if count <= EARLY_MAX_HOLDERS:

                        early_wallets[token].update(
                            current_wallets[token]
                        )


                    history.append(count)


                print(
                    f"\nBlocks "
                    f"{current_block - WINDOW_BLOCKS}"
                    f" -> {current_block}"
                )


                for wallet, score in wallet_scores.items():

                    if score >= SCORE_THRESHOLD:

                        print("👥 Holder Growth Early Wallet")
                        print("Wallet:", wallet)
                        print("Score:", score)
                        print()


                last_block = current_block


            time.sleep(3)


        except Exception as e:

            print("Error:", e)
            time.sleep(5)


if __name__ == "__main__":
    main()
