# Blockchain

#Code Explanation
from hashlib import blake2b
import json
import time

from flask import Flask, request
import requests


class Block:
    def __init__(self, index, transactions, timestamp, previous_hash, nonce=0):  # initialize variables when object created
        self.index = index
        self.transactions = transactions
        self.timestamp = timestamp
        self.previous_hash = previous_hash
        self.nonce = nonce

    def compute_hash(self):
        block_string = json.dumps(self.__dict__, sort_keys=True)   # converts a Python object into a json string.
        return blake2b(block_string.encode()).hexdigest()   # produces digests of any size between 1 and 64 bytes


class Blockchain:
   
    difficulty = 2

    def __init__(self):
        self.unconfirmed_transactions = []
        self.chain = []

    def create_genesis_block(self):
        
        genesis_block = Block(0, [], 0, "0")  # creates object for class Block
        genesis_block.hash = genesis_block.compute_hash() # Calculate hash value and assign to variable hash
        self.chain.append(genesis_block)   # Created object ( genesis_block) is added to list "chain"

    @property
    def last_block(self):
        return self.chain[-1]     # returns the lastly added block in the list "chain"

    def add_block(self, block, proof):
        # adds new block to list "chain"   ------- In this function it checks "whether the block is already added" & " valid Proof"
        # if any one condition fails it return false otherwise it adds block to list "chain" and status return True
        previous_hash = self.last_block.hash

        if previous_hash != block.previous_hash:
            return False

        if not Blockchain.is_valid_proof(block, proof):
            return False

        block.hash = proof
        self.chain.append(block)
        return True

    def proof_of_work(self, block):
        
        block.nonce = 0
        # This function calculates the hash value repetitively  until the hash value starts with "00".
        # returns the hash value starts with "00"
        computed_hash = block.compute_hash()
        while not computed_hash.startswith('0' * Blockchain.difficulty):
            block.nonce += 1
            computed_hash = block.compute_hash()

        return computed_hash

    def add_new_transaction(self, transaction):
        self.unconfirmed_transactions.append(transaction) # adds the transaction to list "unconfirmed_transactions"

    @classmethod
    def is_valid_proof(cls, block, block_hash):
        # This function returns True when "hash value starts with "00" " and "block_hash is equal to computed hash
        
        return (block_hash.startswith('0' * Blockchain.difficulty) and
                block_hash == block.compute_hash())

    @classmethod
    def check_chain_validity(cls, chain):
        # this function checks the chains validity. if the chain is valid then hash and previous hash valus are added and return True otherwise it return False
        result = True
        previous_hash = "0"

        for block in chain:
            block_hash = block.hash
            
            delattr(block, "hash") #deletes an attribute "hash" from the object "block"

            if not cls.is_valid_proof(block, block.hash) or \
                    previous_hash != block.previous_hash:
                result = False
                break

            block.hash, previous_hash = block_hash, block_hash

        return result

    def mine(self):
       
        if not self.unconfirmed_transactions:
            return False

        last_block = self.last_block

        new_block = Block(index=last_block.index + 1,
                          transactions=self.unconfirmed_transactions,
                          timestamp=time.time(),
                          previous_hash=last_block.hash)     # Creaes object for new block

        proof = self.proof_of_work(new_block)            # This function calculates the hash value repetitively  until the hash value starts with "00".
        # returns the hash value starts with "00"

        self.add_block(new_block, proof) # Adds new block to list "chain"


        self.unconfirmed_transactions = []
        announce_new_block(new_block)
        return new_block.index    # returns the index of new block


app = Flask(__name__)

blockchain = Blockchain()
blockchain.create_genesis_block()
peers = set()

@app.route('/new_transaction', methods=['POST'])     #  binds a URL to a function "new_transaction()" with POST method
def new_transaction():
    tx_data = request.get_json() # data get from post method stored in tx_data
    required_fields = ["Patient-ID", "content"]

# if fields "Patient-ID", "content" are not present in the tx_data it generates error Message " Invlaid transaction data"
    # otherwise the transaction is added to block chain and returns Success message
    for field in required_fields:
        if not tx_data.get(field):
            return "Invlaid transaction data", 404

    tx_data["timestamp"] = time.time()

    blockchain.add_new_transaction(tx_data)

    return "Success", 201

@app.route('/chain', methods=['GET'])   #  binds a URL to a function "get_chain()" with GET method
def get_chain():
    chain_data = []
    for block in blockchain.chain:
        chain_data.append(block.__dict__)  # append block to list "chain_data"
    return json.dumps({"length": len(chain_data),
                       "chain": chain_data,
                       "peers": list(peers)})     # converts a Python object into a json string and that string is returned

@app.route('/mine', methods=['GET']) #  binds a URL to a function "mine_unconfirmed_transactions()" with GET method
def mine_unconfirmed_transactions():
    result = blockchain.mine()    # False/index of new bolck
    if not result:
        return "No transactions to mine"
    return "Block #{} is mined.".format(result)

@app.route('/register_node', methods=['POST']) #  binds a URL to a function "register_new_peers()" with POST method
def register_new_peers():  # register with new PEERS
    node_address = request.get_json()["node_address"]   # gets node_address
    if not node_address:
        return "Invalid data", 400

    
    peers.add(node_address)  #node_address is added to peers

    return get_chain()


@app.route('/register_with', methods=['POST'])  #  binds a URL to a function "register_with_existing_peers()" with POST method
def register_with_existing_node():   # register with existing PEERS

    node_address = request.get_json()["node_address"]  # gets node_address
    if not node_address:
        return "Invalid data", 400

    data = {"node_address": request.host_url}
    headers = {'Content-Type': "application/json"}

    
    response = requests.post(node_address + "/register_node",
                             data=json.dumps(data), headers=headers)

    if response.status_code == 200:
        global blockchain
        global peers
        
        chain_dump = response.json()['chain']
        blockchain = create_chain_from_dump(chain_dump)   #new block is created with the values from json
        peers.update(response.json()['peers'])
        return "Registration successful", 200
    else:
        
        return response.content, response.status_code


def create_chain_from_dump(chain_dump):
    generated_blockchain = Blockchain()    # creates object for blockchain
    generated_blockchain.create_genesis_block()   # new block with default vaues is created and hash value is added to list "chain"
    for idx, block_data in enumerate(chain_dump):
        if idx == 0:
            continue  
        block = Block(block_data["index"],
                      block_data["transactions"],
                      block_data["timestamp"],
                      block_data["previous_hash"],
                      block_data["nonce"])   # changing block data to the data we got from json in previous functio
        proof = block_data['hash']
        added = generated_blockchain.add_block(block, proof)
        if not added:
            raise Exception("The chain dump is tampered!!")
    return generated_blockchain



@app.route('/add_block', methods=['POST'])    #  binds a URL to a function "verify_and_add_block()" with POST method
def verify_and_add_block():   # checks whether the block is added or not added to the chain
    block_data = request.get_json()   # getting data
    block = Block(block_data["index"],
                  block_data["transactions"],
                  block_data["timestamp"],
                  block_data["previous_hash"],
                  block_data["nonce"])

    proof = block_data['hash']
    added = blockchain.add_block(block, proof)

    if not added:
        return "The block was discarded by the node", 400

    return "Block added to the chain", 201

@app.route('/pending_tx')   #  binds a URL to a function "get_pending_tx()"
def get_pending_tx():
    return json.dumps(blockchain.unconfirmed_transactions)


def consensus():
    global blockchain

    longest_chain = None
    current_len = len(blockchain.chain)

    for node in peers:
        response = requests.get('{}chain'.format(node))
        length = response.json()['length']
        chain = response.json()['chain']
        if length > current_len and blockchain.check_chain_validity(chain):
            current_len = length
            longest_chain = chain

    if longest_chain:
        blockchain = longest_chain
        return True

    return False


def announce_new_block(block):
 
    for peer in peers:
        url = "{}add_block".format(peer)
        headers = {'Content-Type': "application/json"}
        requests.post(url,
                      data=json.dumps(block.__dict__, sort_keys=True),
                      headers=headers)
