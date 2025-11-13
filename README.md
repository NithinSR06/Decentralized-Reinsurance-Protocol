#![allow(non_snake_case)]
#![no_std]
use soroban_sdk::{contract, contracttype, contractimpl, log, Env, Symbol, String, symbol_short, Address};

// Structure to track the overall pool status
#[contracttype]
#[derive(Clone)]
pub struct PoolStatus {
    pub total_capital: u64,      // Total capital in the pool
    pub active_policies: u64,    // Number of active reinsurance policies
    pub total_claims_paid: u64,  // Total claims paid out
    pub available_capital: u64,  // Capital available for new policies
}

// Structure for individual reinsurance policy
#[contracttype]
#[derive(Clone)]
pub struct ReinsurancePolicy {
    pub policy_id: u64,
    pub insurer: Address,
    pub coverage_amount: u64,
    pub premium_paid: u64,
    pub is_active: bool,
    pub claim_paid: u64,
}

// Storage keys
const POOL_STATUS: Symbol = symbol_short!("POOL");
const POLICY_COUNT: Symbol = symbol_short!("P_COUNT");

// Enum for policy mapping
#[contracttype]
pub enum PolicyBook {
    Policy(u64)
}

#[contract]
pub struct ReinsuranceContract;

#[contractimpl]
impl ReinsuranceContract {
    
    // Function 1: Add capital to the reinsurance pool
    pub fn add_capital(env: Env, amount: u64) -> u64 {
        let mut pool = Self::view_pool_status(env.clone());
        
        pool.total_capital += amount;
        pool.available_capital += amount;
        
        env.storage().instance().set(&POOL_STATUS, &pool);
        env.storage().instance().extend_ttl(5000, 5000);
        
        log!(&env, "Capital added: {}. Total pool capital: {}", amount, pool.total_capital);
        pool.total_capital
    }
    
    // Function 2: Purchase reinsurance policy
    pub fn purchase_policy(env: Env, insurer: Address, coverage_amount: u64, premium: u64) -> u64 {
        let mut pool = Self::view_pool_status(env.clone());
        let mut policy_count: u64 = env.storage().instance().get(&POLICY_COUNT).unwrap_or(0);
        
        // Check if pool has enough capital
        if pool.available_capital < coverage_amount {
            log!(&env, "Insufficient pool capital for coverage");
            panic!("Insufficient pool capital");
        }
        
        policy_count += 1;
        
        let new_policy = ReinsurancePolicy {
            policy_id: policy_count,
            insurer: insurer.clone(),
            coverage_amount,
            premium_paid: premium,
            is_active: true,
            claim_paid: 0,
        };
        
        // Update pool status
        pool.active_policies += 1;
        pool.available_capital -= coverage_amount;
        pool.total_capital += premium;
        
        // Store policy and update pool
        env.storage().instance().set(&PolicyBook::Policy(policy_count), &new_policy);
        env.storage().instance().set(&POOL_STATUS, &pool);
        env.storage().instance().set(&POLICY_COUNT, &policy_count);
        env.storage().instance().extend_ttl(5000, 5000);
        
        log!(&env, "Policy created with ID: {}", policy_count);
        policy_count
    }
    
    // Function 3: File and process claim
    pub fn process_claim(env: Env, policy_id: u64, claim_amount: u64) {
        let mut policy = Self::view_policy(env.clone(), policy_id);
        let mut pool = Self::view_pool_status(env.clone());
        
        // Validate policy and claim
        if !policy.is_active {
            log!(&env, "Policy is not active");
            panic!("Policy not active");
        }
        
        if claim_amount > policy.coverage_amount {
            log!(&env, "Claim exceeds coverage amount");
            panic!("Claim exceeds coverage");
        }
        
        // Process claim
        policy.claim_paid = claim_amount;
        policy.is_active = false;
        
        pool.total_claims_paid += claim_amount;
        pool.active_policies -= 1;
        pool.available_capital += (policy.coverage_amount - claim_amount);
        
        // Update storage
        env.storage().instance().set(&PolicyBook::Policy(policy_id), &policy);
        env.storage().instance().set(&POOL_STATUS, &pool);
        env.storage().instance().extend_ttl(5000, 5000);
        
        log!(&env, "Claim processed for policy {}: {} paid", policy_id, claim_amount);
    }
    
    // Function 4: View pool status
    pub fn view_pool_status(env: Env) -> PoolStatus {
        env.storage().instance().get(&POOL_STATUS).unwrap_or(PoolStatus {
            total_capital: 0,
            active_policies: 0,
            total_claims_paid: 0,
            available_capital: 0,
        })
    }
    
    // Helper function: View specific policy
    pub fn view_policy(env: Env, policy_id: u64) -> ReinsurancePolicy {
        let key = PolicyBook::Policy(policy_id);
        
        env.storage().instance().get(&key).unwrap_or(ReinsurancePolicy {
            policy_id: 0,
            insurer: Address::from_string(&String::from_str(&env, "GAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAWHF")),
            coverage_amount: 0,
            premium_paid: 0,
            is_active: false,
            claim_paid: 0,
        })
    }
}

#[cfg(test)]
mod test {
    use super::*;
    use soroban_sdk::Env;

    #[test]
    fn test_add_capital() {
        let env = Env::default();
        let contract_id = env.register_contract(None, ReinsuranceContract);
        let client = ReinsuranceContractClient::new(&env, &contract_id);
        
        let result = client.add_capital(&1000000);
        assert_eq!(result, 1000000);
    }

}


## Contract details
CDSM5UJAJVB4B4L4CBEAQAGUF6CRLJFVXYDH6ZHNPXB7AF3JL5KLXEOE
![Uploading image.png…]()
